

============
 Pr�ctica 2
============

Log Analyzer System
===================

:Autor: Antonio Camas Maestre
:git: https://github.com/antoniocamas/log-analyzer-architecture.git
   
.. contents:: Contenidos
	      
.. raw:: pdf

   PageBreak
	      
Decisi�n y Justificaci�n de los Patrones Arquitect�nicos
----------------------------------------------------------

El sistema est� divido en dos subsistemas independientes. La comunicaci�n entre ambos ser� a trav�s de una base de datos en la que un subsistema volcar� los logs procesados y el otro los recuperar� para su an�lisis.

Subsistema *Log Processer*
^^^^^^^^^^^^^^^^^^^^^^^^^^

El patr�n *Broker* ordena la estructura general del *Log Processer*. Sin embargo el servidor que procesa los logs presenta varios m�dulos con requisitos espec�ficos que nos llevan a combinar el patr�n *Broker* gobernante con el Patr�n *Layers*. 


Patr�n *Broker*
"""""""""""""""

El subsistema usa el patr�n *Broker* que est� justificado por las siguientes **fuerzas**:

- El sistema debe ser **el�stico** pues debe atender a un numero indefinido de sistemas y aplicaciones generando logs.
- El sistema requiere elasticidad una **arquitectura distribuida** responde bien a esta necesidad. 
- El sistema ha de poder crecer, debemos proveer de mecanismos que abstraigan a los clientes de la localizaci�n de los servidores.

El sistema no es ni tan grande ni tan complicado como para empujarnos a un patr�n de microservicios. Por tanto el patr�n *Broker* es el adecuado para responder a las necesidades.

Como **consecuencia** por el uso  del patr�n Broker el *Log Processer* presenta:

- Transparencia de localizaci�n de los *logs Receptors* (clientes) y de los filtros de logs (servidores).
- Extensibilidad en clientes y servidores.
- Posibilidad de cambiar las implementaciones de sus componentes.
- Portabilidad
- Reusabilidad

Sin embargo esta decisi�n presenta algunas **responsabilidades**:

- Menos eficiente que el monolito.
- Es complicado de probar extremo a extremo.
- Es complicado de depurar de extremo a extremo.
- Aunque en general el patr�n Broker complica la gesti�n de errores. La poca relevancia de perder un dato de entrada hace que en este sistema en particular sea sencillo controlar los errores pues ignorar la linea de log que provoca el fallo es viable.


Patr�n *Layers*
"""""""""""""""

A su vez la parte de ofrece servicios de procesado de logs est� organizada siguiendo un patr�n *Layers*. Las **fuerzas** que empujan a usar este patr�n son:

- Existe la necesidad de tener flexibilidad en el procesamiento de los logs: eliminar, cambiar o a�adir etapas. 
- Agrupaci�n de los componentes con responsabilidades similares (parsers y procesadores del testo por ejemplo).
- Descomposici�n de un componente complejo.

Las **consecuencias** de usar un patr�n *Layers*:

- Podr�a existir una reutilizaci�n de capas
- Dependencias locales: un cambio afecta como mucho a las capas adyacentes
- Las interfaces bien definidas dan la posibilidad de cambiar las implementaciones de los componentes.


Subsistema *Log Analyzer*
^^^^^^^^^^^^^^^^^^^^^^^^^

El subsistema *Log Analyzer* est� gobernado por una arquitectura siguiendo el patr�n *Blackboard*.  Sin embargo la organizaci�n de los analizadores de logs combina el anterior patr�n con el patr�n *Broker* por razones de dimensionamiento, flexibilidad, mantenimiento, etc.

Patr�n *Blackboard*
"""""""""""""""""""

Las **fuerzas** que empujan a usar el patr�n *Blackboard* son:

- Es necesario usar diferentes algoritmos para el an�lisis de los logs.
- Es necesario poder a�adir, modificar o eliminar herramientas de an�lisis.
- Hay algoritmos que resuelven parcialmente el problema.
- Hay resultados aproximados involucrados en el an�lisis.

Las **consecuencias** pues de este patr�n *Blackboard* son:

- el subsistema permite usar diferentes analizadores.
- el subsistema se adaptar� bien a los cambios en los analizadores.
- el esfuerzo en mantenimiento est� acotado.
- los analizadores son reutilizables.
- es tolerante a fallos. Si un analizador falla se ignorar� ignorar� el resultado de su an�lisis.
- es robusto.

Las **responsabilidades**: 

- Dificultad para probar el subsistema completo de extremo a extremo.
- No se pueden garantizar soluciones de calidad.
- Dificultad para dise�ar una buena estrategia de control.
- Poca eficiencia.
- Dificultad en la paralelizaci�n. Algo que se intentar� paliar combin�ndolo con el patr�n *Broker* como se ver� m�s adelante.
- Alto coste de desarrollo.

Patr�n *Broker*
"""""""""""""""
  
Los analizadores del subsistema se disponen siguiendo un patr�n *Broker* debido a las siguientes **fuerzas**:

- El sistema debe ser **el�stico** pues debe atender a un numero indefinido de sistemas y aplicaciones generando logs.
- Flexibilidad en el an�lisis de los logs: poder cambiar el servicio de an�lisis en ejecuci�n.
- Capacidad de a�adir, modificar o eliminar herramientas de an�lisis, sin tener que hacer crecer la aplicaci�n original indefinidamente. Bastar� con agregar, modificar o eliminar servidores.
- Intentar mejorar el rendimiento introduciendo paralelismo.
- Las ventajas de los sistemas distribuidos vistos anteriormente cuando describiamos el subsistema *Log processer* aplican de igual forma aqu�.

Las **consecuencias** y **responsabilidades** son las mismas que al aplicar el mismo patr�n *Broker* en el subsistema *Log Processer*.

Estructura de la soluci�n 
--------------------------

El sistema se estructura en dos subsistemas: *Log Processer* y *Log Analyzer*.

La fase de procesado de logs expone una interfaz para cada tipo de fuente de logs, convierte los logs de distintas fuentes a un formato unificado, eliminando datos superfluos. Guarda los logs procesados (en base de datos) para su posterior an�lisis; La fase de an�lisis obtiene, cada minuto, de la capa de persistencia bloques de logs y los analiza, lanzando una alarma si de su an�lisis se prev� un mal funcionamiento en el sistema monitorizado.

A continuaci�n se describir�n ambos subsistemas relacion�ndolos con los elementos de los patrones arquitect�nicos presentes en el dise�o.


*Log Processer*
^^^^^^^^^^^^^^^

Diagramas:
 - Ver el `Diagrama de Componentes - Log Processer`_.
 - Ver el `Diagrama de Artefactos - Log Processer`_.

Presenta una arquitectura gobernada por el patr�n *Broker*, y est� compuesto por los siguientes elementos:

- Componente *Application Log Receptor*: representa los elementos *Cliente* y *Proxy de Cliente* de patr�n *Broker*. Expone una interfaz para recibir por HTTP los logs de las aplicaciones del sistema monitorizado. Consume la interfaz del componente *Log Processer Broker* encolando los logs recibidos al servicio de procesado de logs.

- Componente *System Log Receptor*: an�logamente representa los elementos *Cliente* y *Proxy de Cliente* de patr�n *Broker*. Exponiendo una interfaz para recibir por HTTP los logs de los elementos sistema.

- Componente *Log Processer Broker*: representa el elemento *Broker* del patr�n. Expone las interfaces para los clientes y el registro de los servidores. Como particularidad de esta soluci�n, los clientes no esperan la respuesta de los servidores, por lo tanto este *broker* no necesita implementar la interfaz del camino de vuelta de los mensajes.

- Servicio *Log Processer*: representa la parte *Servidor* y *Proxy de Servidor* del patr�n Broker. Consume la interfaz de registro de servidores del *broker*. El servicio est� formado por varios componentes organizados por un patr�n *Layers* que aporta flexibilidad al servicio de procesado de logs.

- Componente *App Log Parser*: parte del servicio *Log Processer*, representa parte de la capa *Parser* del patr�n *Layers*. Se encarga de la recepci�n y separaci�n de los campos timestamp, componente, severidad, texto del log. 

- Componente *System Log Parser*: Es an�logo al anterior, es parte del servicio *Log Processer*, representa parte de la capa *Parser* del patr�n *Layers*. Se encarga de la recepci�n y separaci�n de los campos timestamp, componente, severidad, texto del log y mapear la severidad de un entero a las cadenas 1->ERROR, 2->WARN, 3->INFO, 4->DEBUG.

- Componente *Log Body Processer*: parte del servicio *Log Processer*, representa parte de la capa *Body Processer*. Es llamado por los componente de la capa *Parser* y se encarga de separar el texto del log en las diferentes palabras que lo componen. Al terminar llama al componente *Noisy Words Filter*.

- Componente *Noisy Words Filter*: parte del servicio *Log Processer*, representa parte de la capa *Body Processer*. Se encarga de eliminar todas las palabras de 3 o menos caracteres al objeto de facilitar la indexaci�n. Consume la interfaz de la siguiente capa *Persistant Adapter Layer*.

- Componente *Log Recorder*: parte del servicio *Log Processer*, representa parte de la capa *Persistant Adapter Layer*. Hace persistir los logs en el formato unificado en una base de datos.

- Estructura de datos: El formato unificado ser� un documento *json* con la siguiente estructura:


.. code:: json

	  {
	   'timestamp': timestamp,
	   'componente': comp,
	   'severity': sever,
	   'body': [text]
	  }

Vista l�gica  del subsistema *Log Processer*.
"""""""""""""""""""""""""""""""""""""""""""""   
	  
.. figure:: vistas/Dia-Componentest-Log-Processer_v1.png
   :scale: 100%
   :name: Diagrama de Componentes - Log Processer
	 
   Diagrama de Componentes


Vista de despliegue del subsistema *Log Processer*. 
"""""""""""""""""""""""""""""""""""""""""""""""""""

.. figure:: vistas/Dia-Artefactos-Log-Processer_v1.png
   :scale: 100%
   :name: Diagrama de Artefactos - Log Processer
	 
   Diagrama de Artefactos
	  
	  
*Log Analyzer*
^^^^^^^^^^^^^^

Diagramas:

- Ver el `Diagrama de Componentes - Log Analyzer`_.
- Ver el `Diagrama de Artefactos - Log Analyzer`_.

Presenta una estructura dirigida por el patr�n arquitect�nico *Blackboard* combinado con el patr�n *Broker* para dotar a los analizadores de las ventajas de los sistemas distribuidos. Est� compuesto por los siguientes elementos:

- Componente *Control*: representa el elemento *Control* de patr�n *Blackboard*. Se comunica con la pizarra y el representante de los analizadores que ya veremos m�s adelante. Tiene las responsabilidad de tomar la decisi�n sobre cu�ndo hay que comenzar a analizar nuevos logs, elegir los analizadores que entrar�n en juego e interpretar las respuestas de los analizadores para tomar la decisi�n de si el sistema est� entrando en mal funcionamiento o no. 

- Componente *Blackboard*: representa el elemento *Blackboard* de patr�n *Blackboard*. Se trata de un componente que ofrece un espacio de comunicaci�n com�n entre el resto de elementos de subsistema.

- Componente *Analyzer Tool Scheduler*: Es una adaptaci�n del elemento *Algorithm* de patr�n *Blackboard*. A�na la comunicaci�n que los Analizadores que son los verdaderos representantes de los elementos *Algorithm* del patr�n, haciendo las veces de Proxy. Los analizadores se organizan con un patr�n *Broker* del que el *Analyzer Tool Scheduler* hace tambi�n las veces de los elementos *Cliente* y *Proxy de Cliente* del sistema distribuido *Broker*. Se encarga por tanto de recibir las indicaciones de componente *Control*, encolar las peticiones a los analizadores a trav�s del *Analysis Tool Broker* y recibir los resultados de los an�lisis de forma as�ncrona propiciando el paralelismo, y de escribir estos resultados convenientemente en el componente *Blackboard*.

- Componente *Analysis Tool Broker*: representa el elemento *Broker* del patr�n. Expone las interfaces para los clientes y el registro de los servidores. Trabajar� de forma as�ncrona por lo que ofrece interfaces de doble sentido a servidores y clientes. 

- Componentes *Log Comparer Server*, *Language Processer Server* y  *Machine Learning Server*: representan los elementos *Servidor* y *Proxy de Servidor* del patr�n *Broker*. Consumen la interfaz de registro de servidores del *broker*. Reciben paquetes de logs para analizar y devuelven resultados a trav�s del *Analysis Tool Broker*. La especificaci�n interna de los analizadores est� fuera del objetivo de esta descripci�n arquitect�nica. Todos los analizadores pueden trabajar en paralelo y en diferentes m�quinas. Mejorando el rendimiento y aportando elasticidad.

Vista l�gica  del subsistema *Log Analyzer*.
""""""""""""""""""""""""""""""""""""""""""""  
  
.. figure:: vistas/Dia-Componentest-Log-Analyzer_v1.png
   :scale: 100%
   :name: Diagrama de Componentes - Log Analyzer
	 
   Diagrama de Componentes

Vista de despliegue del subsistema *Log Analyzer*. 
"""""""""""""""""""""""""""""""""""""""""""""""""""

.. figure:: vistas/Dia-Artefactos-Log-Analyzer_v1.png
   :scale: 100%
   :name: Diagrama de Artefactos - Log Analyzer
	 
   Diagrama de Artefactos
   
	  
Comportamiento Din�mico de la soluci�n 
---------------------------------------

*Log Processer*
^^^^^^^^^^^^^^^

Ver el `Diagrama de Secuencia - Log Processer - system logs`_ y `Diagrama de Secuencia - Log Processer - Application logs`_.

Vista de procesos del subsistema *Log Processer*. Sistemas
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

.. figure:: vistas/Dia-Secuencia-Log-Processer_System_v1.png
   :scale: 100%
   :name: Diagrama de Secuencia - Log Processer - system logs
	 
   Diagrama de Secuencia para logs de sistema

Vista de procesos del subsistema *Log Processer*. Aplicaciones
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
   
.. figure:: vistas/Dia-Secuencia-Log-Processer_App_v1.png
   :scale: 100%
   :name: Diagrama de Secuencia - Log Processer - Application logs
	 
   Diagrama de Secuencia para logs de aplicaci�n


*Log Analyzer*
^^^^^^^^^^^^^^

Vista de procesos del subsistema *Log Analyzer*.
""""""""""""""""""""""""""""""""""""""""""""""""

.. figure:: vistas/Dia-Secuencia-Log-Analyzer_v1.png
   :scale: 100%
   :name: Diagrama de Secuencia - Log Analyzer
	 
   Diagrama de Secuencia.
