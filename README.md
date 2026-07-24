# QKD v2.0 — BB84 · SSP · E91 con stack crittografico post-quantum

Simulazione in [NetSquid](https://netsquid.org/) di tre protocolli di Quantum Key
Distribution, con la pipeline classica completa: riconciliazione Cascade, decoy
states, privacy amplification e autenticazione post-quantum del canale classico
via [liboqs](https://github.com/open-quantum-safe/liboqs).

Il progetto non si limita a mostrare che BB84 "funziona": misura il costo di ogni
componente della pipeline e verifica che le garanzie di sicurezza dichiarate siano
effettivamente rispettate dal codice.

---

## Indice

- [Cosa contiene](#cosa-contiene)
- [Requisiti](#requisiti)
- [Installazione](#installazione)
- [Esecuzione](#esecuzione)
- [Architettura](#architettura)
- [Parametri principali](#parametri-principali)
- [Scelte progettuali](#scelte-progettuali)
- [Limiti noti](#limiti-noti)
- [Riferimenti](#riferimenti)

---

## Cosa contiene

**Tre protocolli QKD**

| Protocollo | Paradigma | Basi | Sifting | Soglia QBER | Test di sicurezza |
|---|---|---|---|---|---|
| BB84 | prepare-and-measure | 2 (Z, X) | ~50% | 11.0% | QBER |
| SSP (six-state) | prepare-and-measure | 3 (Z, X, Y) | ~33% | 12.6% | QBER |
| E91 | entanglement-based | 3 angoli | ~22% | 11.0% | CHSH (S > 2) |

**Stack classico**

- **Cascade completo** (Brassard & Salvail 1994) — 4 passate, blocco adattivo
  `k₁ = ⌊0.73/QBER⌋`, propagazione ricorsiva delle correzioni. Confrontato con
  una baseline a 2 passate per quantificarne il costo in bit divulgati.
- **Decoy states** (Hwang 2003; Lo-Ma-Chen 2005) — bound LMC sul key rate con
  sorgente laser attenuata, contromisura al Photon-Number-Splitting.
- **Privacy Amplification** — Toeplitz hashing con tetto del Leftover Hash Lemma.
- **AES-256-GCM** — cifratura finale con la chiave derivata.

**Stack post-quantum (liboqs)**

- **ML-DSA-65** (FIPS 204) — firma dei messaggi del canale classico.
- **ML-KEM-768** (FIPS 203) — componente K1 dello scambio ibrido.
- Chiave di sessione: `K = HKDF-SHA256(K1 ‖ K2_QKD, session_id)`.

**Modelli fisici**

Attenuazione in fibra (Beer-Lambert), PMD, dark counts, jitter temporale dei
rivelatori (SNSPD vs InGaAs), con decomposizione del QBER per sorgente.

**Attacchi simulati**

- Intercept-Resend con tasso di intercettazione variabile (η).
- Man-in-the-Middle attivo su canale quantistico **e** classico.

---

## Requisiti

- Python 3.11+
- **NetSquid** — richiede registrazione gratuita su [netsquid.org](https://netsquid.org/)
  (non è su PyPI pubblico)
- `numpy`, `scipy`, `matplotlib`, `cryptography`
- `liboqs-python` (opzionale ma raccomandato — vedi sotto)
- `pyspx` (fallback per la firma se liboqs non è disponibile)

---

## Installazione

```bash
python -m venv venv_netsquid
source venv_netsquid/bin/activate

pip install numpy scipy matplotlib cryptography pyspx
pip install netsquid --extra-index-url https://pypi.netsquid.org  # richiede credenziali
```

### liboqs

> **Attenzione all'omonimia su PyPI.** Il pacchetto `oqs` è *Open Quick Script*,
> un interprete di espressioni **senza alcun legame** con la crittografia
> post-quantum. Entrambi installano un modulo importabile come `oqs` e si
> sovrascrivono a vicenda. Il pacchetto corretto è `liboqs-python`.

```bash
pip uninstall -y oqs            # rimuove l'omonimo, se presente
pip install liboqs-python
python -c "import oqs; print(oqs.oqs_version())"
```

Se l'ultimo comando fallisce con `No oqs shared libraries found`, la libreria C
non è stata compilata. Serve `cmake`:

```bash
pip install cmake ninja
rm -rf ~/_oqs                   # rimuove il tentativo fallito
python -c "import oqs; print(oqs.oqs_version())"
```

Se la compilazione si ferma su `Could NOT find OpenSSL`, mancano gli header di
sviluppo. Due strade:

```bash
# A — installa gli header (richiede sudo)
sudo apt install libssl-dev

# B — compila senza OpenSSL (nessun sudo)
git clone --depth=1 --branch 0.15.0 \
    https://github.com/open-quantum-safe/liboqs.git liboqs-src
cmake -S liboqs-src -B liboqs-build -GNinja \
      -DBUILD_SHARED_LIBS=ON -DOQS_BUILD_ONLY_LIB=ON -DOQS_USE_OPENSSL=OFF \
      -DCMAKE_INSTALL_PREFIX=$HOME/.local
ninja -C liboqs-build && ninja -C liboqs-build install
export LD_LIBRARY_PATH=$HOME/.local/lib:$LD_LIBRARY_PATH
```

Con la strada B, `LD_LIBRARY_PATH` va esportato **prima** di lanciare
Jupyter o VS Code: un `export` dentro una cella non ha effetto sul kernel
già avviato.

### Senza liboqs

Il notebook resta eseguibile: la firma ricade su pyspx (SPHINCS+) e il KEM su
X25519. In quel caso i risultati **non sono post-quantum su K1**, e il codice lo
dichiara esplicitamente — sia a runtime (`quantum_safe_k1=False`) sia nel testo
delle conclusioni, che passa automaticamente a una versione con avviso.
`print_oqs_diagnostic()` stampa la causa esatta del fallback.

---

## Esecuzione

Il progetto è **una singola cella** in `progettobb84_v3_unificato.ipynb`. Aprire
il notebook ed eseguirla.

L'esecuzione completa richiede alcuni minuti e produce:

- output testuale di tutte le fasi (FASE 0 → FASE 8)
- `/tmp/qkd_physical_model.png` — 4 pannelli sul modello fisico
- `/tmp/qkd_dashboard_final.png` — dashboard a 12 pannelli

### Uso programmatico

```python
safe, key, stats = run_qkd_protocol(
    num_bits=3000,
    distance=25.0,
    noise_rate=0.03,
    protocol="BB84",          # "BB84" | "SSP" | "E91"
    eve_present=False,
    use_full_cascade=True,
    cascade_passes=4,
    use_sphincs_auth=True,    # firma PQ del canale classico
    use_decoy_states=True,
    use_mlkem_hybrid=True,    # K = HKDF(K1_MLKEM ‖ K2_QKD)
    jitter_detector="SNSPD",  # "SNSPD" | "InGaAs"
    pa_strict=False,          # True = rifiuta sessioni con <128 bit estraibili
)
```

`stats` contiene QBER grezzo e post-riconciliazione, bit divulgati, output della
privacy amplification (`pa_output_bits`, `pa_available_bits`), e i sotto-dizionari
`decoy`, `sphincs`, `mlkem`.

---

## Architettura

| Modulo | Contenuto |
|---|---|
| 1 | Fisica del canale: attenuazione, PMD, dark counts, jitter |
| 2A-D | Protocolli BB84/SSP + Eve Intercept-Resend |
| 2E | E91: sorgente entangled, misure ad angolo, parametro CHSH |
| 2F | Attacco MITM attivo (quantistico + classico) |
| 3 | Cascade completo + baseline a 2 passate |
| 3B | Decoy states, bound LMC, analisi PNS |
| 3C | Firma post-quantum (liboqs) + benchmark + test di correttezza |
| 3D | Hybrid KEM ML-KEM-768 + derivazione HKDF |
| 4 | Privacy Amplification (Toeplitz, tetto LHL) |
| 5 | Runner unificato BB84/SSP/E91 |
| 6 | Cifratura AES-256-GCM |
| 7 | Analisi No-Cloning + sicurezza E91 |
| 8A-8H | Trade-off distanza, statistiche, key rate, confronto PA, modello fisico |
| 9 | Dashboard a 12 pannelli |
| 10 | Conclusioni narrative (condizionate sui dati misurati) |

---

## Parametri principali

| Costante | Valore | Note |
|---|---|---|
| `FIBRE_ATTENUATION_DB_PER_KM` | 0.1419 | fibra PSCF ultra-low-loss |
| `DARK_COUNT_RATE_HZ` | 10.0 | SNSPD stato dell'arte |
| `JITTER_SNSPD_PS` | 50.0 | contributo al QBER trascurabile a 1 GHz |
| `JITTER_INGAAS_PS` | 400.0 | ~10.6% di QBER a 1 GHz — fattore dominante |
| `CLOCK_PERIOD_NS` | 1.0 | sistema a 1 GHz |
| `MU_SIGNAL` / `MU_DECOY1` | 0.5 / 0.1 | intensità segnale e decoy |
| `SIG_ALG_PRIMARY` | `ML-DSA-65` | `SLH-DSA-SHAKE-128s` come alternativa |
| `KEM_ALG_PRIMARY` | `ML-KEM-768` | fallback X25519 senza liboqs |

Il seed globale (`set_global_seed`) rende riproducibili le scelte di base e le
perdite fotoniche. Le chiavi crittografiche **non** sono seedate: usano sempre il
CSPRNG del sistema, perché chiavi deterministiche sarebbero un antipattern di
sicurezza.

---

## Scelte progettuali

### ML-DSA-65 invece di SLH-DSA

Il benchmark integrato misura, sulla macchina in uso, ~3309 B di firma contro
~7856 B e ~0.17 ms contro ~294 ms. In QKD l'overhead del canale classico non è
un dettaglio: con SLH-DSA la sola autenticazione costerebbe ~15.3 KB per
sessione, un ordine di grandezza più della chiave prodotta.

**Prezzo dichiarato:** con ML-DSA-65 per la firma e ML-KEM-768 per il KEM,
entrambe le componenti post-quantum poggiano sui reticoli. Le loro sicurezze
**non sono indipendenti**. Resta indipendente K2 (QKD), che è
information-theoretic. Per diversificare le assunzioni basta impostare
`SIG_ALG_PRIMARY = "SLH-DSA-SHAKE-128s"`.

### Privacy Amplification con tetto LHL

L'output è limitato a `n_raw − bit_divulgati − security_param`. Le sessioni senza
entropia sufficiente vengono **rifiutate** invece di produrre una chiave
apparentemente valida; sotto i 256 bit l'espansione ad AES avviene via HKDF ed è
dichiarata (HKDF non crea entropia).

Conseguenza sperimentale: per una chiave AES-256 con garanzia piena servono ≥256
bit estraibili, cioè ≥320 bit siftati puliti. Da qui l'uso di 2000-3000 impulsi
per sessione anziché poche centinaia.

### Verifica del MITM in entrambe le direzioni

Il blocco del MITM non è dedotto dal solo caso felice ("la firma legittima è
valida"), ma dal rifiuto effettivo delle due strategie a disposizione di Eve:
riuso della firma di Alice su un messaggio diverso, e firma con chiavi proprie.
`verify_signature_soundness()` gira in FASE 0 e verifica che la firma rifiuti
messaggi manomessi e chiavi pubbliche errate.

---

## Limiti noti

- **Propagazione Cascade troncata a un livello** per motivi di prestazioni: una
  minoranza di sessioni (~17-24% a QBER 3-5%) conserva un piccolo numero di
  errori residui. Il valore è misurato a runtime e riportato in FASE 6D, non
  ereditato da stesure precedenti.
- **Overhead Cascade sistematico solo in media.** Sulla singola sessione una
  frazione dei run mostra il contrario, perché il numero di errori da correggere
  fluttua. Con pochi bit siftati 2 passate possono già bastare.
- **Riduzione del key rate diluita dal cap della PA.** Sopra i 256 bit estraibili
  l'overhead in bit non si traduce in perdita di key rate, e alle distanze lunghe
  il rate è già nullo: la media su tutte le distanze sottostima l'effetto.
- **QBER additivo con clamp a 50%.** I contributi sono sommati (approssimazione
  valida per QBER piccoli). Oltre il 50% il valore è troncato, perché un QBER
  superiore è fisicamente privo di significato; il flag `qber_saturated` segnala
  quando il clamp interviene e il modello additivo ha già perso validità.
- **E91 su lunga distanza** soffre di una perdita strutturale: due tratte da d/2
  implicano `P(coppia) = P(d/2)²`.
- **Sicurezza device-independent solo in linea di principio.** La violazione di
  Bell qui misurata non è loophole-free.

---

## Riferimenti

- Bennett, C. H., & Brassard, G. (1984). *Quantum cryptography: Public key
  distribution and coin tossing.* Proc. IEEE Int. Conf. on Computers, Systems and
  Signal Processing, 175–179.
- Ekert, A. K. (1991). *Quantum cryptography based on Bell's theorem.* Physical
  Review Letters, 67(6), 661–663.
- Brassard, G., & Salvail, L. (1994). *Secret-key reconciliation by public
  discussion.* EUROCRYPT '93, LNCS 765, 410–423.
- Hwang, W.-Y. (2003). *Quantum key distribution with high loss: Toward global
  secure communication.* Physical Review Letters, 91, 057901.
- Lo, H.-K., Ma, X., & Chen, K. (2005). *Decoy state quantum key distribution.*
  Physical Review Letters, 94, 230504.
- Ma, X., Qi, B., Zhao, Y., & Lo, H.-K. (2005). *Practical decoy state for quantum
  key distribution.* Physical Review A, 72, 012326.
- Clauser, J. F., Horne, M. A., Shimony, A., & Holt, R. A. (1969). *Proposed
  experiment to test local hidden-variable theories.* Physical Review Letters,
  23(15), 880–884.
- Hadfield, R. H. (2009). *Single-photon detectors for optical quantum information
  applications.* Nature Photonics, 3, 696–705.
- NIST FIPS 203 (ML-KEM), FIPS 204 (ML-DSA), FIPS 205 (SLH-DSA), 2024.
- Krawczyk, H., & Eronen, P. (2010). *HKDF: HMAC-based Extract-and-Expand Key
  Derivation Function.* RFC 5869.
