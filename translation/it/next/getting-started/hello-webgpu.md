Ciao WebGPU <span class="bullet">🟢</span>
============

```{lit-setup}
:tangle-root: 001 - Ciao WebGPU - Next
:parent: 000 - Project setup - Next
:fetch-files: ../../data/webgpu-distribution-v0.3.0-alpha.zip
```

*Resulting code:* [`step001-next`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step001-next)

WebGPU è una *Render Hardware Interface* (RHI), ovvero una libreria di programmazione volta a fornire un'**interfaccia unificata** per molteplici configurazioni hardware e sistemi operativi.

Per il codice C++, WebGPU è incapsulato in un **singolo file header**, che lista tutte le procedure e strutture dati disponibili: [`webgpu.h`](https://github.com/webgpu-native/webgpu-headers/blob/main/webgpu.h)

Tuttavia, quando compiliamo un programma, il tuo compilatore deve sapere (durante lo step di *Linkaggio*) **dove trovare** l'implementazione vera e propria di queste funzioni. A differenza delle API native, l'implementazione di WebGPU non è fornita dal driver, quindi dobbiamo fornirla esplicitamente.

```{figure} /images/rhi-vs-opengl.png
:align: center
Una *Render Hardware Interface* (RHI) come WebGPU **non è fornita dai driver**: dobbiamo linkarla alla libreria che implementa l'API sopra la libreria a basso livello che il sistema supporta.
```

Installare WebGPU
-----------------

Ci sono due implementazioni dell'header di WebGPU:

 - [wgpu-native](https://github.com/gfx-rs/wgpu-native), basato sulla libreria in Rust [`wgpu`](https://github.com/gfx-rs/wgpu), che non solo supporta Firefox ma anche una grande porzione di applicazioni grafiche in Rust.
 - Google's [Dawn](https://dawn.googlesource.com/dawn), l'implementazione di WebGPU usata da Chromium e i suoi derivati. (Google Chrome, MS Edge, etc.).

```{figure} /images/different-backend.png
:align: center
Ci sono almeno due implementazioni dell'header di WebGPU, sviluppate per i due motori web principali.
```

```{note}
> Un'implementazione di WebGPU che non è supportata in questo caso è quella dal [WebKit](https://webkit.org/). Potrebbe essere aggiunta in futuro, anche se non è una priorità siccome non è cross-platform. (non supporta Windows).
```

Queste due implementazioni hanno delle **discrepanze**, ma scompariranno man mano che la specifica di WebGPU diventa stabile. Cercherò di scrivere questa guida in modo che **funzioni per entrambe**


**Per facilitare l'integrazione** in un progetto CMake, ho condiviso una repository [WebGPU-distribution](https://github.com/eliemichel/WebGPU-distribution) che ti permette di scegliere i dettagli attraverso variabili CMake, ma espone la stessa interfaccia qualunque implementazione tu decida di usare.

```{important}
Quando guardiamo gli **esempi** provvisti in ciascuna pagina della guida, controlla la cartella `webgpu` per vedere su **quale versione della distribuzione** essi si basano. WebGPU è in continua evoluzione quindi ogni aggiornamento potrebbe rompere delle cose.
```

### Integrazione

Il modo più facile per integrare questa distribuzione è **copiare il suo contenuto nel tuo sorgente** (sono solo un paio di file CMake):

 1. **Scarica** l'archivio [zip di WebGPU-distribution](https://github.com/eliemichel/WebGPU-distribution/archive/refs/tags/main-v0.3.0-alpha.zip).
 2. **Estrailo** nella root del tuo progetto e chiamalo `webgpu`. Questa cartella dovrebbe contenere un file `CMakeLists.txt` (se non lo include, rimuovi le cartelle extra innestate).
 3. Aggiungi `add_subdirectory(webgpu)` nel tuo `CMakeLists.txt`.

```{lit} CMake, Dependency subdirectories (insert in {{Define app target}} before "add_executable")
# Includi la cartella di webgpu per definire il target 'webgpu'
add_subdirectory(webgpu)
```

```{important}
il nome `webgpu` designa la **cartella** dove è situata la nostra distribuzione di webgpu, quindi dovrebbe esserci un file `webgpu/CMakeLists.txt`. Altrimenti significa che quel `webgpu.zip` non è stato estratto nella cartella corretta; potresti doverla spostare o addattare la direttiva `add_subdirectory`.
```

 4. Aggiungi il target `webgpu` come una **dipendenza** della tua app, usando il comando `target_link_libraries` (dopo `add_executable(App main.cpp)`)
    

```{lit} CMake, Link libraries (insert in {{Define app target}} after "add_executable")
# Aggiungi il target 'webgpu' come dipendenza della nostra App
target_link_libraries(App PRIVATE webgpu)
```

```{tip}
Questa volta, il nome 'webgpu' è uno dei **target** definiti in `webgpu/CMakeLists.txt` chiamando `add_library(webgpu ...)`, non è relativo al nome della cartella.
```

 5. Un passo aggiuntivo è richiesto quando usiamo il **linking dinamico** (esempio, quando il backend di WebGPU è distribuito come un file .so/.dll/.dylib accanto al tuo eseguibile): **chiama la funzione**
    `target_copy_webgpu_binaries(App)` alla fine del file `CMakeLists.txt` per assicurarti che i file .so/.dll/.dylib siano copiato accanto ad esso.

```{note}
Quando **distribuisci** la tua applicazione, assicurati di distribuire anche questi file dinamici delle librerie!
```

```{lit} CMake, Link libraries (append)
# Il binario dell'applicazione deve trovare i file .so/.dll/.dylib durante l'esecuzione
# perciò copiamo automaticamente i file accanto all'esegubibile
target_copy_webgpu_binaries(App)
```

```{tip}
Nel caso del linkaggio statico (l'oposto del linkaggio dniamico), la funzione `target_copy_webgpu_binaries` è sempre definita (cosi che tu non debba adattare il tuo `CMakeLists.txt`) ma non fa nulla.
```

### CMake options

Le opzioni di CMake e le variabili di cache sono definite per attivare **prendi una versione specifica del backend**. Potresti saltare questa sezione se non ti importa, in generale le opzioni di CMake possono essere specificare nel terminale quando chiamiamo CMake:

```bash
# Chiama CMake con il valore 'IL_MIO_VALORE' assegnato alla variaible `MIA_OPZIONE`
cmake -B build -DMY_OPTION=MY_VALUE
```

#### Scelta di implementazione

La prima variabile che vuoi cambiare è `WEBGPU_BACKEND`, che può essere `WGPU`, `DAWN`, o `EMSCRIPTEN`

```{tip}
Quando usi `emcmake` (il wrapper CMake fornito da emscripten), **non c'è bisogno** di impostare esplicitamente `WEBGPU_BACKEND` su `EMSCRIPTEN`. Sarà automaticamente individuato e nessuna implementazione verrà recuperata.
```

#### Compilare dal sorgente

Di base, la distribuzione recupera una **versione precompilata** dell'implementazione di WebGPU cosi che il tuo progetto compili più velocemente. Se preferisci compilare dal sorgente, imposta l'opzione `WEBGPU_BUILD_FROM_SOURCE` su `ON`. Richiederà più tempo e dipendenze extra (Python nel caso di Dawn).

```{note}
Puoi compiare dal sorgente **solo con Dawn** per ora. Siccome wgpu-native è scritto in rust, la sua integrazione nel nostro processo di compilazione C++ è più complessa.
```
**Per più opzioni**, e più dettagli riguardo cosa potrebbe motivare le loro scelte, ti invito a visitare il [README di WebGPU-distribution](https://github.com/eliemichel/WebGPU-distribution). Nel frattempo, **consiglio** di usare le versioni pre-compilate prima, con Dawn o wgpu-native.

#### Esempi

In sintesi, ci sono un paio di esempi su come personalizzare la tua build:

```bash
# Compila usando una versione pre-compilata del backend di wgpu-native
cmake -B build-wgpu -DWEBGPU_BACKEND=WGPU -DWEBGPU_BUILD_FROM_SOURCE=OFF
cmake --build build-wgpu

# Compila usando un backend Dawn compilato dal sorgente
cmake -B build-dawn -DWEBGPU_BACKEND=DAWN -DWEBGPU_BUILD_FROM_SOURCE=ON
cmake --build build-dawn

# Compila usando emscripten (non c'è bisogno di un backend specifico -- vedi sotto se non conosci emscripten)
emcmake cmake -B build-emscripten
cmake --build build-emscripten
```

### Comportamento di implementazioni specifiche

Questa guida intende fornire codice che è **compatibile con tutti i backend**. Siccome ci sono ancora piccole differenze tra le implementazioni, le distribuzioni che fornisco definiscono le seguendi direttive del preprocessore:

```C++
// Se usi Dawn
#define WEBGPU_BACKEND_DAWN

// Se usi wgpu-native
#define WEBGPU_BACKEND_WGPU

// Se usi emscripten
#define WEBGPU_BACKEND_EMSCRIPTEN
```

Testare l'installazione
------------------------

Per testare l'implementaizone, creiamo **un'istanza** di WebGPU (equivalente al `navigator.gpu` in JavaScript). Poi lo controlliamo e lo distruggiamo.

```{lit} C++, file: main.cpp
{{Includes}}

int main (int, char**) {
    {{Crea l'istanza di WebGPU}}

    {{Controlla l'istanza di WebGPU}}

    {{Rilascia l'istanza di WebGPU}}

    return 0;
}
```

```{important}
Assicurati di includere `<webgpu/webgpu.h>` prima di usare qualsiasi funzione o tipo di WebGPU!
```

```{lit} C++, Includes
// Includes
#include <webgpu/webgpu.h>
#include <iostream>
```

### Descrittori e Creazione

L'istanza è creata usando la funzione `wgpuCreateInstance`. Vedremo che tutte le funzioni di WebGPU pensate per **creare** un'entità prendono come argomento un **descrittore**. Questo descrittore è usato per specificare le opzioni riguardanti a come impostiamo questo oggetto.

```{lit} C++, Create WebGPU instance
// Creiamo un descrittore
WGPUInstanceDescriptor desc = {};
desc.nextInChain = nullptr;

// Creiamo l'istanza che usa il descrittore
WGPUInstance instance = wgpuCreateInstance(&desc);
```

```{note}
Il descrittore è un modo per **impacchettare più argomenti di funzione** insieme, perché alcuni descrittori hanno molti campi. Può anche essere usato per scrivere funzioni di utilità che pensano a popolare gli argomenti, per semplificare l'architettura del programma.
```

Qui incontriamo un altro **idioma** di WebGPU nella struttura `WGPUInstanceDescriptor`: il primo campo del descrittore è sempre un puntatore chiamato `nextInChain`. E' un modo generico per l'API di attivare **estensioni personalizzate** per aggiungerle in futuro, o per ritornare molteplici voci di dati. **Nella maggiorparte dei casi, lo impostiamo a `nullptr`.** 


### Controllo

Un'entità di WebGPU creata con una funzione `wgpuCreaQualcosa` è tecnicamente **solo un puntatore**. E' un gestore opaco che identifica l'oggetto vero e proprio, che vive nel backend e a cui non accediamo mai direttamente.

Per controllare che un oggetto sia valido, lo confrontiamo con `nullptr`, o usiamo l'operatore booleano:

```{lit} C++, Check WebGPU instance
// Controlliamo se c'è un'istanza creata.
if (!instance) {
    std::cerr << "Could not initialize WebGPU!" << std::endl;
    return 1;
}

// Stampa l'oggetto (WGPUInstance è un puntatore, potrebbe
// essere copiato in giro senza preoccuparci della sua dimensione.).
std::cout << "WGPU instance: " << instance << std::endl;
```

Questo dovrebbe mostrare qualcosa come `WGPU instance: 000001C0D2637720` all'avvio.

### Distruzione e gestione emivita

Tutte le entità che sono state **create** usando WebGPU a un certo punto devono essere **rilasciate**. Una procedura che crea un oggetto ha una dicitura del tipo `wgpuCreateSomething`, e l'equivalente per rilasciarlo è `wgpuSomethingRelease`.

Nota che ogni oggetto ha internamente un contatore dei riferimenti, e rilasciare l'oggetto libera soltanto la memoria relativa al riferimento puntato. (il contatore viene decrementato fino a 0):

```C++
WGPUSomething sth = wgpuCreateSomething(/* descriptor */);

// Significa "aumenta il contatore dei riferimenti dell'oggetto sth a 1"
wgpuSomethingAddRef(sth);
// Ora i riferimenti sono 2  (è impostato a 1 durante la creazione)

// Questo significa "decrementa il contatore dei riferimenti dell'oggeto sth di 1
// se arriva a 0 distruggi l'oggetto.
wgpuSomethingRelease(sth);
// Ora il contatore è tornato a 1, l'oggetto può ancora essere usato.

// Rilascia di nuovo
wgpuSomethingRelease(sth);
// Ora i riferimenti sono a 0, l'oggetto è distrutto e
// non dovrebbe essere più usato!
```

In particolare, dobbiamo rilasciare l'istanza globale di WebGPU:

```{lit} C++, Release WebGPU instance
// Puliamo l'istanza di WebGPU
wgpuInstanceRelease(instance);
```

### Compilare per il Web

Le distribuzioni di WebGPU sopra indicate sono compatibili con [Emscripten](https://emscripten.org/docs/getting_started/downloads.html) e se hai problemi nel compilare l'applicazione per il web, puoi consultare [l'appendice indicato](../appendices/building-for-the-web.md).

Man mano che aggiungeremo alcune opzioni specifiche per la compilazione web, possiamo aggiungere una sezione alla fine del nostro `CMakeLists.txt`:

```{lit} CMake, file: CMakeLists.txt (append)
# Opzioni specifiche per Emscripten
if (EMSCRIPTEN)
    {{Emscripten-specific options}}
endif()
```

Per ora cambiamo solo l'estensione dell'output cosi da avere una pagina web HTML (piuttosto che un modulo WebAssembly o una libreria JavaScript):

```{lit} CMake, Emscripten-specific options
# Genera una pagina web piuttosto che un modulo WebAssembly
set_target_properties(App PROPERTIES SUFFIX ".html")
```

Per qualche motivo l'istanza del descrittore **deve essere null** (che vuol dire "usa il default") quando usiamo Emscripten, cosi possiamo usare la nostra macro `WEBGPU_BACKEND_EMSCRIPTEN`:


```{lit} C++, Create WebGPU instance (replace)
// Creiamo un descrittore
WGPUInstanceDescriptor desc = {};
desc.nextInChain = nullptr;

// Creiamo l'istanza che usa questo descrittore
#ifdef WEBGPU_BACKEND_EMSCRIPTEN
WGPUInstance instance = wgpuCreateInstance(nullptr);
#else //  WEBGPU_BACKEND_EMSCRIPTEN
WGPUInstance instance = wgpuCreateInstance(&desc);
#endif //  WEBGPU_BACKEND_EMSCRIPTEN
```

Conclusione
----------

In questo capitolo abbiamo impostato WebGPU e abbiamo appreso che ci sono **molteplici backend** disponibili. Abbiamo visto anche gli idiomi di base per la **creazione e distruzione di oggetti** che verranno usati tutto il tempo nell'API di WebGPU!

*Codice finale:* [`step001-next`](https://github.com/eliemichel/LearnWebGPU-Code/tree/step001-next)
