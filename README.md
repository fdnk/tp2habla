# Proyecto 2 - Sistema de reconocimiento de habla en español

## Sobre Bash
### Ejecutar tareas en paralelo
En este TP se hace uso extensivo de programas que reciben una lista de archivos de la forma `<archivo de entrada> <archivo de salida>`. Para aprovechar la capacidad de cómputo disponible, se utiliza GNU Parallel para correr tantas tareas en paralelo como hilos haya en las CPU.

**Ejemplo práctico:**  Adaptando este ejemplo se puede correr sin problemas los programas como HCopy en paralelo:

Sea el archivo fl:
```
one uno
two dos
three tres
five cinco
```
Ejecutando el (los) comandos:

```$ time parallel -a fl -j+0 --eta --colsep ' '  'echo {2} - {1}'```

Se obtiene:
```
Computers / CPU cores / Max jobs to run
1:local / 4 / 4

Computer:jobs running/jobs completed/%of started jobs/Average seconds to complete
one - uno
two - dos
three - tres
five - cinco
ETA: 0s 0left 0.00avg  local:0/4/100%/0.0s

real	0m0.373s
user	0m0.164s
sys	0m0.044s
```

## Estructura de directorios
Se crea la siguiente estructura de directorios dentro de `~/proyectos`.
```
cd ~/proyectos
mkdir proyecto2 && cd proyecto2
mkdir datos etc modelos rec log config scripts lm doc
```

### Inicializaciones
Para hacer más cómodo el trabajo de aquí en más se definen las siguentes variables dentro del archivo mypaths y se lo corre con `export mypaths`. Recordar hacer esto cada vez que se trabaje con el TP2 o bien, agregarlo al .bashrc

```
# Setea las variables con las rutas para hacer más sencillo el trabajo
# Use source mypaths para emplear dichos atajos

export TP=~/proyectos/proyecto2
export DOC=/home/cestien/proyectos/doc
export DB=/dbase/latino40
```

Configuro el charset del entorno
```
echo export LC_CTYPE=ISO_8859_1 >> ~/.bashrc
echo export LESSCHARSET="iso8859" >> ~/.bashrc
source ~/.bashrc
```
Y para ver los carácteres en la consola, setearla en *Western (ISO_8859_1)*.

Linkeo los `.wav` y copio los archivos con datos
```
cd $TP
ln -s $DB/wav datos/
cp $DB/doc/promptsl40.train $DB/doc/promptsl40.test etc/
cp $DOC/lexicon etc/
```

Me copio los scripts
```
cp $DOC/go.* $DOC/prompts2mlf $TP/scripts/
```
## Creación del diccionario
1. Me armo una lista de palabras, tanto de train como de test, y elimino repetidas
```
cd $TP/etc
cat promptsl40.train promptsl40.test | awk '{for(i = 2; i <= NF; i++) print $i}' | uniq > wlistl40
```

2. En base al lexicon, extraigo las palabras que liste anteriormente.

En la configuración de HDMan, el archivo `$TP/etc/global.ded`, guardo los comandos:
```
AS sp
MP sil sil sp
```
3. Y corro HDMan. De paso, me guarda una lista con los monofonos usados en `monophones`.
```
cd $TP/etc
HDMan -m -w wlistl40 -n monophones -l $TP/log/dlog dictl40 lexicon
```

Por alguna extraña razón, la codificación de los carácteres de dictl40 queda mal. Para subsanar este problema, se realizaron los siguientes reemplazos, guardando el archivo como ISO_8859_1:
| Código | Reemplazo |
|------|---|
| \341 | á |
| \351 | é |
| \355 | í |
| \363 | ó |
| \372 | ú |
| \374 | ü |
| \361 | ñ |


## Parametrización de los datos
Se utiliza HCopy para obetener los `.mfc`. Se emplea la siguiente configuración, que se guarda en el archivo `$TP/config/config.hcopy` (haciendo `cp $DOC/config.hcopy $TP/config/`).
```
# Coding parameters
TARGETKIND = MFCC_0 # Con C0
#TARGETKIND = MFCC_0_D_A
TARGETRATE = 100000.0
SAVECOMPRESSED = T
SAVEWITHCRC = T
WINDOWSIZE = 250000.0
USEHAMMING = T
PREEMCOEF = 0.97
NUMCHANS = 26
CEPLIFTER = 22
NUMCEPS = 12   # 12 coefs
ENORMALISE = F # Sin nommalización de energía
SOURCEFORMAT = NIST
```

Armo la estructura de directorios y la lista de archivos a procesar
```
cd $TP/datos
mkdir -p mfc/train mfc/test  # Creo los directorios
$TP/scripts/go.mfclist trainmfc testmfc # Armo la lista de archivos
```
Voy a utilizar GNU Parallel para ejecutar varias instancias de HCopy en paralelo para aprovechar los recursos de las CPUs. Guardo un log en `$TP/log/hcopy` para no ensuciar la consola. Es importante usar >> para que las distintas instancias de HCopy no borren el log de las otras.
```
date > $TP/log/hcopy
time parallel -j+0 --eta 'HCopy -T 1 -C $TP/config/config.hcopy -S {} >> $TP/log/hcopy' ::: trainmfc testmfc
```

En mi caso, el resultado de la ejecución es el siguiente:
```
nando@nando-HR14:~/habla/proyectos/proyecto2/datos$ time parallel -j+0 --eta 'HCopy -T 1 -C $TP/config/config.hcopy -S {} >> $TP/log/hcopy' ::: trainmfc testmfc

Computers / CPU cores / Max jobs to run
1:local / 4 / 2

Computer:jobs running/jobs completed/%of started jobs/Average seconds to complete
ETA: 102s 0left 52.00avg  local:0/2/100%/52.0s s

real	1m44.365s
user	1m47.524s
sys	0m1.612s
```

## Creación de MLF de transcripciones de palabra y fonéticas
Para armar el MLF de palabras, creo una lista con todos los MFC y uso el script `prompts2mlf`:
```
cd $TP/etc
cat promptsl40.train promptsl40.test > promptsl40
$TP/scripts/prompts2mlf wordmlf promptsl40
```
Para armar el MLF de fonemas se usa HLed. En la configuración se hace que borre los sp para crear el primer modelo de monofonos usando el siguiente `mkphones0.led`:
```
EX
IS sil sil
DE sp
```
Y luego `mkphones1.led` no elimina los sp:
```
EX
IS sil sil
```

Creo los fonos en cada caso:
```
HLEd -l '*' -d dictl40 -i phones0.mlf mkphones0.led wordmlf  # Sin sp
HLEd -l '*' -d dictl40 -i phones1.mlf mkphones1.led wordmlf  # Cpn sp
```

## Creación de modelos de monofonos con una sola gausiana
Los modelos fonéticos se crearan basados en el prototipo `$DOC/proto`. El mismo crea un modelo de cinco estados, tres emisores y dos dos no emisores con coeficientes de velocidad y acelarción y C0.

El modelo se entrena con la media y la varianza global de todas las muestras de entrenamiento usando HCompV

Configuración general: `$TP/config/config`
```
TARGETKIND = MFCC_0_D_A
```

Preparo la lista de archivos y copio el proto
```
cp $DOC/proto $TP/etc/
find $TP/datos/mfc/train | grep "\.mfc" > $TP/etc/mfc_trainlist
```

Y corro HCompV sobre los archivos de entrenamiento
```
cd $TP/modelos
mkdir hmm0
HCompV -C $TP/config/config -f 0.01 -m -S $TP/etc/mfc_trainlist -M hmm0 $TP/etc/proto
```

De la vez que use HDMan, conseguí una lista de monofonos en `monophones`. En dicha lista, voy a incluir **sil**, porque no se encuetra, voy a eliminar **sp** para hacer el primer modelo.
```
cd $TP/etc
cp monophones monophones1
echo sil >> monophones1  # sil y sp
grep -v "sp" monophones1 > monophones0 # sil

```

Creo hmmdefs y macros:
```
cd $TP/modelos/hmm0
$TP/scripts/go.gen-hmmdefs $TP/etc/monophones0 proto > hmmdefs
$TP/scripts/go.gen-macros vFloors proto > macros
```

Reestimo el modelo. Itero la estimacion un par de veces.
```
cd $TP/modelos
for i in {0..2}
do
   mkdir -p hmm$((i+1))
   HERest -C $TP/config/config -I $TP/etc/phones0.mlf -t 250.0 150.0 1000.0 -S $TP/etc/mfc_trainlist -H hmm$((i))/macros -H hmm$((i))/hmmdefs -M hmm$((i+1)) $TP/etc/monophones0
done
```

Creé hmm4 y copié el estado central de sil a sp. Uso un modelo de 3 estados. Pongo la matriz de transición correspondiente e dimensiones... total, después se va a estimar bien.
Uso HHEd para crear los lazos.
```
cd $TP/modelos
mkdir -p hmm5
HHEd -H hmm4/macros -H hmm4/hmmdefs -M hmm5 $TP/config/sil.hed $TP/etc/monophones1
```
Y ahora HERest de vuelta, considerando los sp:
```
cd $TP/modelos
for i in {4..6}
do
   mkdir -p hmm$((i+1))
   HERest -C $TP/config/config -I $TP/etc/phones1.mlf -t 250.0 150.0 1000.0 -S $TP/etc/mfc_trainlist -H hmm$((i))/macros -H hmm$((i))/hmmdefs -M hmm$((i+1)) $TP/etc/monophones1
done
```

## Creación de los modelos de lenguaje

## Reconocimiento con el modelo de unigramas y una sola gausiana

## Refinamiento del modelo agregando mezclas de gausianas

## Implementación de la gramática finita.
