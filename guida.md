# Guida a un'integrazione felice

## Struttura del progetto

Il progetto si struttura in tre parti principali:

1. `calc`
2. `itp_components`
3. `refinement`
4. `main.elpi`

La prima e l'ultima parte sono quelle "invasive", cioe' sono le parti in cui il
progetto va letteralmente a modificare quanto c'era prima. Le altre due sono
parti modulari che semplicemente si inseriscono a lato e non modificano il
comportamento.

### `calc`

La prima parte riguarda la modifica dei giudizi di tipaggio e di conversione
precedenti. In particolare sono state fatte due cose:

1. modifica di `isa` da giudizio ternario a quaternario (i.e. aggiunta del
   raffinamento)
2. modifica di `conv`, in particolare l'uso di `subst` anziche' avere
   applicazione diretta

La prima modifica e' necessaria per implementare il raffinamento, inoltre e'
stato modificato il corpo delle clausole per gestire metavriabili.

```[prolog]
of (app Lam X) CX (app Lam' X') IE :-
    isa Lam (setPi B C) Lam' IE,
    isa X B X' IE,
    pi x\ (locDecl x B, copy x X') => conv (C x) CX.
```

Come si vede da questo snippet oltre al raffinamento e' stato inserito un
giudizio di conversione, questo per tutti i tipi dipendenti. Il giudizio di
conversione tenta di prevenire la fuoriuscita dal frammento L$_\lambda$. In
particolare, quando abbiamo un tipo dipendente, come in questo caso, il tipo
ritornato e' l'applicazione di un tipo a uno o piu' argomenti, che
tendenzialmente saranno termini arbitrari. Per ovviare a questo si utilizza la
`conv` con apposite ipotesi che serviranno di fatto durante l'unificazione (
`conv` chiamata su termini che contengono meta arriviera' all'unificazione).

In questo caso le ipotesi aggiunte sono di due tipi e vengono sempre inserite
sotto una o piu' quantificazioni universali:

1. dichiarazioni `locDecl ci Ci` che dichiarano che la variabile fresca `ci` e'
   del tipo `Ci`,
2. dichiarazioni di copia di una variabile fresca in una metavariabile i.e.
   `copy ci Xi`.

Le prime servono per eventuali richieste di type inference o type checking sulle
variabili fresche, le seconde vengono usate durante la proiezione.

Lo schema di queste ipotesi e' semplice: per ogni termine a cui un tipo
dipendente sarebbe applicato si crea una variabile fresca, la si tipa del tipo
corretto - eventualmente applicato a variabili fresche che stanno per altre
metavariabili - e si mette un'ipotesi di copia. Queste ipotesi implicano il
giudizio di conversione in cui compare il tipo dipendente applicato solo a
variabili fresche. Questa "tecnica" e' analoga a quella impiegata nel giudizio
`subst` e serve per far rientrare cose esterne al frammento in cose nel
frammento.

La modifica ai giudizi di conversione potrebbe non essere una cosa non
particolarmente utile, bisogna valutare cosa passa per la conversione i.e. se
quando si chiama `conv` e si arriva a quelle regole si hanno solo termini chiusi
(no metavariabili) allora si e' a cavallo, altrimenti si rischia di uscire dal
frammento.

Per l'integrazione di questa parte bisogna letteralmente modificare i giudizi
precedenti.

### `itp_components`

Questa parte e' quella che costituisce il dimostratore interattivo e si compone
di `itp.elpi` e la cartella `itp_components`, divisa in `internals.elpi` e
`io.elpi`.

Il primo file contiene varie definizione di predicati per lanciare il
dimostratore, il cui entry point e' `itp_loop`. Questi predicati sono
`prove_term`, `prove_goals` e `prove`, di cui il principale e' `prove` e gli
altri sono semplicemente altri modi di far partire (con goal multipli, con
termini).

L'idea di `prove` e' semplicemente di lanciare il tipaggio con
`of X Type ProofTerm` e poi chiamare il loop creando a mano il goal iniziale.
Volendo si potrebbero creare macro che astraggono la creazione di goal iniziali
per evitare di sbagliarsi.

#### Internals

In questo file sono contenute le definizioni vere e proprie per il dimostratore
interattivo i.e. la definizione del loop, la gestione dei tinycals etc.

L'entry point e' `itp_loop` che e' un predicato con 5 "argomenti":

1. il termine di input, cioe' il primo termine su cui si e' lanciato prove
2. lo stato corrente
3. lo script di input
4. i goal eventualmente rimanenti al termine della dimostrazione es. goal che
   non si e' riuscito a chiudere oppure interruzione della dimostrazione prima
   della completa fine
5. lo script di output

Il termine in input iniziale serve per i controlli di raggiungibilita' delle
metavariabili prima di chiudere il loop: alcuni goal possono rimanere aperti ma
possono corrispondere a metavariabili irraggiungibili es. `((x\ zero) Y)`
genererebbe un goal per `Y` che pero', qualora venisse eseguita la
$\beta$-riduzione, andrebbe persa. Pertanto, prima di chiudere il loop, viene
eseguito un controllo di raggiungibilita' che, a partire dal termine in input,
identifica quelle che sono le meta effettivamente raggiungibili dal termine e
rimuove le altre.

Il secondo argomento e' lo stato della dimostrazione e non viene mai toccato
direttamente dal loop ma semplicemente dai tinycals. Questo ha la forma:

```[prolog]
state [ActiveGoals] [[InactiveGoals]] [levels] MaxGoalNumber
```

`[ActiveGoals]` e' una lista che contiene i goal correntemente attivi ed e'
formata da coppie `(Id, Goal)` dove `Id` e' un intero che indica il numero del
goal (i goal sono numerati progressivamente in ordine di come sono creati).
`Goal` e' a sua volta un termine della forma

```[prolog] 
goal Term TypeOfTerm RefTerm VDecls
```

Dove:

* `Term` e' una metavariabile
* `TypeOfTerm` e' una rappresentazione del tipo di `Term`
* `RefTerm` e' una metavariabile a cui `Term` si raffina
* `VDecls` e' una lista di dichiarazioni di variabile della forma
  `vdecl Var TypeOfVar`, dove `Var` e' un atomo che corrisponde un nome di
  variabile e `TypeOfVar` e' il suo tipo

`Term`, `TypeOfTerm` e `RefTerm` compaiono in un constraint della forma

```[prolog]
isa Term TypeOfTerm RefTerm -> suspended on [Term]
```

Quindi, di fatto, un goal non e' altro che la rappresentazione dei constraint di
tipaggio posti automaticamente dalla chiamata del predicato `of` su un termine
contenente metavariabili.

#### IO

Questa componente e' quella probabilmente meno utile perche' semplicemente si
occupa dello stampare a schermo le informazioni contenute negli stati i.e.
rappresentazione dei goal, dei contesti etc. Si appoggia sull'esistenza del
pretty printing per i termini della logica che si vuole rappresentare.

### `refinement`

### `main.elpi`

L'ultima parte riguarda la modifica dei giudizi in `main.elpi`, in particolare

* sospensione del tipaggio nei casi con metavariabile
* aggiunta di definizione con dimostrazione (con e senza script)

Anche questa e' una modifica "invasiva" perche' va a modificare codice
precedente.

La sospensione del tipaggio si vede in

```[prolog]
of      (uvar as X) T Y IE :- declare_constraint (isa X T Y IE) [X, Y].
ofType  (uvar as X) K   IE :- declare_constraint (ofType X K IE) [X].
isa     (uvar as X) T Y IE :- declare_constraint (isa X T Y IE) [X, Y].
isaType (uvar as X) K   IE :- declare_constraint (isaType X K IE) [X].
```

La definizione con dimostrazione e'

```[prolog]
% Caso di definizione senza termine -> lo devi dimostrare
process_entry (locDefL N TY) Hyp :-
    process_entry (locDefL N TY (script [])) Hyp.

% Caso di prova con uno script (completo o meno) in input
process_entry (locDefL N TY (script S)) Hyp :-
    spy(verify_arg_order N),
    spy(isaType TY _ int),
    erase_var_app TY TY',
    prove TE TY' RTE S _,
    Hyp = locDef N TY RTE,
    print "defined " N TY RTE,
    print "## New definition: " N.
```

Questo significa che si puo' chiamare `process_entry` su una cosa come (vede in
fondo a `test.elpi`)

```[prolog]
(univPi A\ locDefL
    setId
    (setPi A (_\ A))
    (script [intro hyp, app hyp]))
```

E questo di fatto fara' partire il dimostratore che alla fine genera il termine
che va della `locDef` e quindi nell'ipotesi finale.