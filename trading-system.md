# Sistema di Trading Algoritmico

Le modifiche rispetto alla versione originale sono marcate con 🔧 e una breve motivazione.
Questo documento presuppone che prima di costruire l'infrastruttura completa venga eseguita una validazione di metodologia su 1-2 titoli (vedi sezione finale "Sequencing").

> **Decisione di progetto — Orizzonte target: 3 giorni lavorativi**
> Basandosi sulla letteratura quantitativa (momentum a breve termine, Jegadeesh & Titman e successivi), l'orizzonte di 3 giorni lavorativi rappresenta il miglior compromesso tra:
> - Edge documentato su small/mid cap
> - Costi di transazione contenuti rispetto al movimento atteso (2-4%)
> - Affidabilità del backtest (slippage irrilevante rispetto al rendimento atteso)
> - Minor competizione con algoritmi HFT e market maker
>
> 1 giorno è ancora molto contestato (modelli overnight ben arbitraggiati). Oltre 5 giorni il segnale tecnico si degrada e si entra in competizione con strategie fondamentali.

---

## PIPELINE ETL (ogni minuto se il mercato è aperto)

1. **Acquisisci i dati minuto per minuto** (Apertura, Massimo, Minimo, Chiusura, Volume, etc.) — fase E
2. **Analizza i dati, aggiungi nuove feature** (anche da dati storici nel DB se occorre) — fase T

   🔧 **Aggiunta feature di microstruttura, non solo OHLCV.**
   OHLCV puro a 1 minuto è il segnale più "consumato" del mercato — è probabile che qualunque pattern ricavabile da queste sole variabili sia già arbitraggiato da chi ha infrastrutture più veloci. Se disponibili, aggiungi:
   - Order flow imbalance (anche approssimato, se hai accesso a book L2)
   - Deviazione da VWAP
   - Volatilità realizzata multi-scala (1min / 5min / 15min / 1 giorno)
   - Feature cross-asset: movimento del settore/indice nello stesso intervallo
   - 🔧 **Nuovo rispetto alla versione 1min:** feature di momentum multi-giorno (rendimento degli ultimi 2/5/10 giorni), media mobile a 5 e 10 giorni, ATR a 3 giorni — questi segnali sono più informativi sull'orizzonte target di 3 giorni rispetto alle feature puramente intraday

3. **Carica su DB** — fase L

4. Se un titolo raggiunge **100.000 righe** può avviare il flusso MLOps per quel titolo (modello specifico per quel titolo)

   ⚠️ *Nota non risolta, solo segnalata:* 100.000 righe a 1 minuto ≈ 390 righe/giorno di trading ≈ 256 giorni (quasi un anno) prima che un titolo nuovo possa iniziare l'addestramento. Verifica se è accettabile o se abbassare la soglia con un trade-off esplicito su qualità del modello.

5. Ogni volta che un titolo aumenta di altri **50.000 righe**, ripete la pipeline MLOps.

---

## PIPELINE MLOPS (1 volta al giorno, fuori orario di mercato)

1. Ordinamento Cronologico
2. Resampling — 🔧 **Modifica:** il dato grezzo rimane a 1 minuto per le feature di microstruttura, ma viene aggregato anche a risoluzione **giornaliera** per le feature di momentum multi-giorno usate come input al modello
3. Computazione dei Valori Mancanti
4. Codifica categoriale (se occorre)

5. 🔧 **Split sostituito: da Time-Based Split singolo a Walk-Forward con Purging + Embargo.**
   Un singolo split 60/20/20 dà una sola stima di performance, facilmente overfittabile. Usa invece walk-forward validation con più finestre temporali consecutive di train/test che avanzano nel tempo, con:
   - **Purging:** rimuovi dal train le osservazioni il cui orizzonte di label si sovrappone al periodo di test (su 3 giorni questo è particolarmente critico — un'etichetta calcolata su 3 giorni futuri può facilmente "sconfinare" nel periodo di test)
   - **Embargo:** lascia un buffer temporale di almeno 3 giorni lavorativi tra fine train e inizio test (pari all'orizzonte del target)
   - Un modello si considera "pronto" solo se supera le soglie di validazione su più finestre consecutive

6. 🔧 **Target: Triple-Barrier Labeling a orizzonte 3 giorni lavorativi.**
   Predire il prezzo direttamente è fuorviante (random walk). Usa il metodo delle tre barriere (López de Prado):
   - **Barriera superiore = take-profit:** 2× ATR(3 giorni) sopra il prezzo di ingresso — su 3 giorni un movimento del 2-4% è realistico
   - **Barriera inferiore = stop-loss:** 1× ATR(3 giorni) sotto il prezzo di ingresso
   - **Barriera verticale = timeout:** 3 giorni lavorativi (circa 1.170 minuti di mercato aperto)
   - L'etichetta è quale barriera viene toccata per prima: **BUY** (superiore), **SELL** (inferiore), **HOLD/neutro** (timeout)

   🔧 Lo scaling mobile (normalizzazione su max/min degli ultimi 10 giorni lavorativi, coerente con l'orizzonte più lungo) va calcolato solo con dati fino al momento della normalizzazione — mai includendo dati futuri, per evitare leakage silenzioso.

7. **Addestramento di tre modelli — BiLSTM + Attention (Stateless), XGBoost, TFT** — con l'obiettivo di predire la classe (BUY/SELL/HOLD) del triple-barrier a 3 giorni, più stima di volatilità.

   🔧 **Aggiungi, se possibile, un secondo modello di meta-labeling:** un primo modello predice la direzione, un secondo predice se conviene prendere il trade (probabilità di successo calibrata). Questo fornisce la W calibrata necessaria per Kelly.

   🔧 **Nota sull'orizzonte più lungo:** su 3 giorni il numero di campioni di training è circa 3× inferiore rispetto all'orizzonte 1 ora (stesso storico, etichette non sovrapponibili). Verifica che il numero di trade etichettati per titolo sia sufficiente per addestrare in modo affidabile — in caso contrario abbassa la soglia di 100.000 righe o estendi lo storico richiesto.

8. **Fase di test — criteri di validazione:**

   🔧 *Aggiunta correzione per test multipli (deflated Sharpe / soglie più severe).* Con N titoli × 3 modelli stai testando centinaia di combinazioni in parallelo — alcune supereranno i criteri per puro rumore statistico.

   - Directional Accuracy > 53-55%
   - F1-Score > 0.55 (sulle classi BUY/SELL/HOLD)
   - 🔧 Le soglie devono essere superate su più finestre walk-forward consecutive, non su un singolo test
   - 🔧 Calibra la probabilità di output del modello (Platt scaling o isotonic regression) — necessario per ensemble sensato e per Kelly

   Se rispettati → salva il modello, stato "pronto per backtest"

9. **Backtesting — criteri:**

   - 🔧 **Deflated Sharpe Ratio > 1.5** (Bailey & López de Prado) — corregge lo Sharpe osservato per il numero di trial effettuati. Su orizzonte 3 giorni il numero di trade per titolo è inferiore (≈ 60-80 trade/anno per titolo), quindi la stima dello Sharpe ha più incertezza — usa intervalli di confidenza espliciti
   - Maximum Drawdown < 10-15%
   - Profit Factor > 1.2
   - 🔧 I costi di transazione nel backtest devono includere slippage stimato in funzione del volume dell'ordine. Su 3 giorni il peso dei costi è molto inferiore rispetto all'orizzonte 1 ora, ma va comunque modellato
   - 🔧 **Benchmark esplicito:** confronta sempre con buy&hold e con una strategia random a pari turnover. Su orizzonte 3 giorni il buy&hold è un benchmark più difficile da battere di quanto sembri — verificarlo prima è non negoziabile

   Se superato → modello attendibile

10. Per ogni titolo: 3 modelli + scaling per l'encoding dell'output

---

## PIPELINE TRADING SYSTEM (ogni minuto per monitoraggio, decisioni 1-2 volte al giorno)

🔧 **Modifica architetturale rispetto alla versione 1 minuto:** il sistema continua a leggere dati ogni minuto (per aggiornare feature e monitorare stop loss), ma le **decisioni di ingresso** vengono prese **1-2 volte al giorno** (es. pre-apertura e metà giornata), non ad ogni minuto. Questo riduce i falsi segnali e i costi di transazione mantenendo la reattività necessaria per la gestione del rischio.

1. Per ogni titolo con almeno un modello addestrato, **previsione a 3 giorni lavorativi.**
   Con più modelli pronti, uso l'**Ensemble (media pesata per confidenza)**

   🔧 La media pesata per confidenza ha senso solo se le probabilità dei tre modelli sono calibrate allo stesso modo — altrimenti stai mediando numeri non comparabili e l'ensemble può risultare peggiore del modello singolo migliore.

   Output in tabella forecast: titolo, prezzo/volume attuale, direzione prevista a 3 giorni, probabilità calibrata, W calibrata (dal modello di meta-labeling)

2. 🔧 **Ordinamento per Valore Atteso / K% di Kelly**, al netto di costi di transazione e slippage stimato in funzione del volume dell'ordine relativo al volume medio giornaliero (su 3 giorni il rapporto costi/rendimento è molto più favorevole, ma va comunque calcolato):

$$K\% = W - \frac{1-W}{R} \quad \text{dove } R = \frac{TP\%}{SL\%}$$

   Scarta subito i titoli con K% ≤ 0 (valore atteso negativo).

3. **Gestione della posizione aperta — Trailing Stop su orizzonte 3 giorni:**

   Su un orizzonte più lungo il trailing stop cambia natura rispetto alla versione 1 minuto:
   - Monitora ogni minuto, ma agisce solo su movimenti significativi (≥ 0.5× ATR giornaliero)
   - **Soglia con isteresi:** per mantenere la posizione oltre il secondo giorno, la probabilità calibrata di continuazione deve superare la soglia di ingresso + 5 punti percentuali (es. se soglia ingresso era 55%, per estendere richiedi >60%)
   - **Cap massimo:** massimo 1 estensione oltre i 3 giorni originali (massimo 4 giorni lavorativi totali)
   - **Definizione operativa di stazionarietà:** il prezzo resta entro ± 0.3× ATR(giornaliero) per 2 giorni consecutivi → considera uscita anticipata (HOLD che non si muove consuma capitale e opportunità)
   - **Stop Loss dinamico basato su ATR:** 1.5× ATR(3 giorni) dall'ingresso, mai allargato, può essere stretto progressivamente man mano che la posizione è in profitto

4. **Kelly Criterion per il sizing:**

$$K\% = W - \frac{1-W}{R}$$

   🔧 Usa **Kelly frazionario (1/4 di K%)**, non Kelly pieno. Su 3 giorni hai meno trade e quindi stime di W meno stabili rispetto a un sistema ad alta frequenza — la penalità per sovrastimare W è asimmetrica (drawdown gravi). 1/4 Kelly è conservativo ma difendibile finché non hai almeno 200+ trade reali per titolo.

   🔧 Usa la **W calibrata per-trade** (dal modello di meta-labeling, non la Directional Accuracy storica aggregata).

5. **Innesco ordine** con quantità calcolata (1/4 Kelly frazionario), orizzonte target 3 giorni lavorativi, Stop Loss basato su 1.5× ATR(3 giorni)

6. **Ordine di acquisto/vendita eseguito**

---

## STATI DEI TITOLI

| Stato | Descrizione |
|---|---|
| **IN_ACQUISIZIONE** | Dati insufficienti per l'addestramento |
| **IN_ADDESTRAMENTO** | Dati sufficienti, addestramento in corso (invalida modelli precedenti) |
| **IN_TRADING_BASE** | 1 modello valido |
| **IN_TRADING_INTERMEDIO** | 2 modelli validi |
| **IN_TRADING_AVANZATO** | 3 modelli validi |
| *(ritorno)* | Se nessun modello valido generato → ritorna a IN_ACQUISIZIONE |

---

## STATI DEGLI ORDINI

**BUY** — **HOLD** (modifica timer/estensione) — **SELL**

---

## Sequencing consigliato (non opzionale)

L'infrastruttura sopra è corretta come disegno, ma non sostituisce una validazione di metodologia su piccola scala. **Costruire prima la pipeline completa è il modo più rapido per sprecare mesi di lavoro su una metodologia che non funziona.**

### Step 1 — Esperimento minimo (prima di costruire qualsiasi infrastruttura)
- 1-2 titoli liquidi, 12-18 mesi di dati storici
- Triple-barrier labeling con orizzonte 3 giorni
- XGBoost semplice (non LSTM/TFT), walk-forward purged con embargo di 3 giorni
- Feature: solo OHLCV + volatilità realizzata + rendimento multi-giorno
- Confronto esplicito con: buy&hold, strategia random a pari turnover
- **Criterio di successo:** edge netto dei costi documentato e stabile su almeno 3 finestre walk-forward consecutive

**Se l'esperimento minimo non mostra edge → il problema è nella metodologia o nel segnale, non nella scala. Tre reti neurali per titolo non lo risolvono.**

### Step 2 — Solo se Step 1 positivo: costruisci la pipeline completa
Segui l'architettura descritta sopra.

### Step 3 — Paper trading (minimo 2-3 mesi)
- Confronta risultati live vs backtest per individuare discrepanze (latenza, slippage reale, dati mancanti, gap overnight)
- Su orizzonte 3 giorni, 2-3 mesi di paper trading producono ≈ 20-30 trade per titolo — numero minimo per una stima preliminare di W reale
- Solo dopo paper trading positivo → capitale reale, partendo dal sizing minimo possibile

---

## Note finali di rischio sistemico

1. **Gap overnight e weekend:** su orizzonte 3 giorni sei esposto a gap di apertura (earnings, macro, geopolitica) che nessun trailing stop intraday può proteggere. Considera un limite di esposizione massima per titolo e per settore.
2. **Regime change:** i modelli addestrati su un regime di bassa volatilità performano male in regime di alta volatilità e viceversa. Aggiungi un rilevatore di regime (es. HMM a 2 stati su VIX o volatilità realizzata dell'indice) che sospende il trading quando il regime corrente è troppo diverso dal regime di training.
3. **Correlazione tra titoli in portafoglio:** Kelly per-titolo non considera la correlazione tra le posizioni aperte simultaneamente. In un drawdown di mercato, posizioni su titoli diversi cadono insieme — il Kelly aggregato del portafoglio può essere molto più aggressivo di quanto i K% individuali suggeriscano.
