---
published: false
title: Idealista en Swift!
layout: post
---
# idealista ya está 100% en Swift!

El paso a Swift de cualquier aplicación, sea del tamaño que sea, es algo trivial en mano de cualquier desarrollador de iOS aunque tenga poca experiencia, ya que es un lenguaje fácil de aprender y que se adapta muy bien portándolo desde Objective-C.

No te creas todo lo que te digan. En algunos casos si puede cumplirse, quizá si tu aplicación consiste en un listado con un detalle y ya está tardes un día en migrarla. Si la aplicación es relativamente grande pero está bien estructurada, quizá tardes un par de meses en la migración. Sin embargo si la aplicación es muy grande, está muy acoplada y tiene mucho *legacy code*, preparate para sufrir, llorar y patalear porque van a ser unos meses en los que vas a producir cero y vas a estresarte mil.

Desde que Apple anuncio el 2 de Junio de 2014 el nuevo lenguaje, en idealista estuvimos trasteando y aprendiendo su sintaxis para poder empezar a usarlo lo antes posible. Desde ese momento decidimos que todas las nuevas funcionalidades las escribiríamos en Swift y que poco a poco iríamos migrando los ficheros de Objective-C a Swift con el objetivo de, tarde o temprano, tener toda la aplicación en el nuevo lenguaje de Apple.

La primera clase de Swift en producción estuvo en la versión **....** en diciembre de 2014\. Al principio todo era maravilloso: Xcode nos creó el bridging header y empezamos a ver que dependencias teníamos con ficheros de Objetive-C. Aquí encontramos el primer problema: al estar todo el *core* de la aplicación en Objective-C teníamos muchísimas dependencias: sistemas de traducciones, formateadores de texto, validaciones, delegados, el *App Delegate*...

El *bridging-header* cada vez crecía más y cada vez que creabamos nuevas funcionalidades en Swift encontrabamos nuevas dependencias. El problema al final no era que el *bridging header* fuese enorme, sino que la traducción automática que se hacía de los ficheros de Objective-C a Swift no existía el concepto de **optionals**, por lo que todo lo que veíamos desde el código de Swift de Objective-C se trataba como **implicitly unwrapped optionals** (es decir, con !). Esto generaba problemas a la hora de usar los parámetros, ya que no quedaba claro cuales eran **nullables** y cuales no (y por lo tanto era una fuente de **crashes**).

A partir del **Xcode 6.3** Apple permitío meter anotaciones en Objective-C que permitían indicar la **nulabilidad?** de los parámetros y las properties de Objective-C. En este momento nos planteamos dos alternativas:

* **Repasar todos los ficheros de Objective-C e ir comprobando su nulabilidad y etiquetando**
* **No hacer nada y esperar a que la clase esté en Swift**

La primera alternativa suponía un gran esfuerzo, ya que por esta fecha aún teníamos más del 50% del proyecto en Objective-C. Sin embargo, la segunda suponía arriesgarse a tener crashes en *runtime* por no utilizar opcionales. Al final llegamos a una conclusión: los objetos que se usasen desde Objective-C y que fuesen delicados (sobre todo respuestas de la API que podían venir con campos sin rellenar) si los etiquetaríamos; el resto no y los pasaríamos cuanto antes a Swift. La idea funcionó bastante bien aunque en alguna versión se nos escapó algún crash de *BAD\_ACCESS*; nada destacable.

Una vez que cogimos la costumbre de desarrollar las nuevas funcionalidades en **Swift** comenzamos a realizar los **test unitarios** también en Swift. Aquí encontramos varios problemas:

1. No veíamos los ficheros de Swift en los test.
Para poder ver los ficheros, era necesario añadirlos al **target** de Test o bien marcarlos como públicos. En nuestro caso, decidimos añadirlos al target.
2. No teníamos manera de mockear nuestros objetos excepto a mano.
Nos tocó crear a mano los **stubs/mocks**, lo que era una tarea pesada (y aburrida) pero necesaria.

Una vez que Apple liberó **Swift 2** y el **Xcode 7**, se solucionó el problema con los test al poder utilizar *@testable import*. Tuvimos que modificar los ficheros para quitarlos del target de test y añadir el import en todos los test; parecía que la cosa iba mejorando. Sin embargo, algunos test que teníamos hechos en Objective-C de clases de Swift no compilaban, ya que eran incapaces de ver las clases ya que el testable import solo está disponible en Swift. Tuvimos que comentar todos estos test y migrarlos al nuevo lenguaje; más trabajo.

A medida que ibamos teniendo más conocimiento de *Swift* y de sus *value types* empezamos a pensar cómo utilizarlos en el código. Las **estructuras y su inmutabilidad, junto con el paso por valor en vez de por referencia** encajaban perfectamente con objetos de dominio de la aplicación, aportándonos mayor seguridad. También los nuevos enumerados y la posibilidad de asociarles valor, junto con la posibilidad de darles "superpoderes" podía aportarnos valor a la aplicación. Sin embargo, había un problema: todas estas nuevas estructuras no eran compatibles con Objective-C por lo que utilizarlas en el código no era viable a no ser que esa parte de la aplicación esté completamente en Swift. Otro problema más que tendríamos que solucionar en el futuro.

Todos los problemas vistos hasta ahora eran molestos pero se podían solventar de forma más o menos sencilla, ya que al final era cuestión de ir migrando cosas al nuevo lenguaje de forma modularizada (siempre pasar cosas relacionadas para evitar las interdependencias entre ficheros de Objective-C y Swift) pero nos encontramos con un problema que no tenía solución: **compilación extremadamente lenta**. En las primeras versiones de Swift, cada vez que se tocaba un fichero era necesario compilar todo, ya que no existía el concepto de **compilación progresiva**. Esto no era un problema si la compilación fuese rápida pero en nuestro caso y dado el tamaño del proyecto una compilación de cero tardaba entre 10 y 20 minutos. Era **desesperante**.

Con Swift 1.2 y una de las betas del Xcode 6 (no recuerdo bien cuál de ellas) se arregló este problema, añadiendo compilación progresiva, lo que eliminó en parte el problema aunque si compilabamos de cero seguía tardando sus 15 minutos. Sin embargo, cuando se tocaba una clase de Objective-C que estaba en el bridging header (que en ese momento, eran muchas) tocaba recompilar todos los ficheros de Swift. A medida que ibamos avanzando con la migración este problema se iba mitigando pero mientras tanto la producción bajaba en picado debido a los tiempos que había que esperar a que el proyecto compilase. Este problema a día de hoy no lo hemos podido solucionar.

Otro (gran) problema que tuvimos fue que teníamos que soportar **iOS 7** y por lo tanto no era posible utilizar frameworks dinámicos, lo que provocaba que no pudiesemos usar **pods** en Swift ni crear nuestros propios módulos para mejorar los tiempos de compilación y encapsular funcionalidades de la aplicación. Este problema era grave, pero no mucho; lo que si que fue grave e hizo imposible seguir soportando iOS 7 fue un bug introducido por Apple en la actualización 7.3 del Xcode y Swift 2.2, que provocaba que en iOS 7 las clases genéricas no funcionasen correctamente [Bug Report] (<https://bugs.swift.org/browse/SR-815>).

Por último, destacar otros problemas menores como que el autocompletado se le vaya la olla y no funcione durante un buen rato, que lldb en las primeras versiones de Swift no nos diese información de las variables a la hora de debuggear o las supuestas migraciones "automáticas" entre versiones de Swift que nos hacían pasar de unos pocos errores de compilación a varios cientos.

Como podéis ver ha sido un camino largo y tortuoso. Sin embargo, si volviese atrás volvería a hacerlo. Todos los problemas que hemos encontrado nos han valido para tener que esforzarnos en aprender la nueva sintaxis, profundizar en las tripas del lenguaje y buscarnos la vida para solventar los baches que muchas veces imponía. El seguir desde el principio la evolución del lenguaje ha permitido visualizar hacía donde se dirige Swift y poder adaptar nuestro código a las nuevas características del lenguaje, haciendo la aplicación más **swifty**. Está claro que Swift ha venido para quedarse y al liberar el código Apple (bravo!!!) miles de personas quieren colaborar, lo que provocará una evolución brutal adaptándolo a las necesidades de los programadores que lo usan diariamente.

El resultado de la migración al nuevo lenguaje es una reducción de un 25% del número de líneas del proyecto, menos crashes en producción gracias al fuerte tipado de Swift, un mayor uso de patrones de diseño, reducción de la deuda técnica al migrar código antiguo, ser **felices por no utilizar más Objective-C y su sintaxis** y hemos aprendido nuevos lenguajes gracias a los lentos tiempos de compilación (no hay mal que por bien no venga!).

Por lo tanto, si estas planteandote migrar tu aplicación a Swift, piensatelo dos veces pero si das el paso te convertirás en mejor persona :-)