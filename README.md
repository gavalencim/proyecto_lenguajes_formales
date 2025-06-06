#!/usr/bin/env python3
"""
# First & Follow Analyzer (LL(1) and SLR(1) Parser)

## Overview
This Python script implements a syntactic analyzer supporting both LL(1) and SLR(1) parsing techniques. It reads a context-free grammar from a text file, computes First and Follow sets, constructs parsing tables, and analyzes input strings to determine if they are accepted. The code is designed for educational purposes with detailed Spanish comments for clarity and debugging.

## Features
- Reads grammar rules and input strings from a text file.
- Computes First and Follow sets for LL(1) and SLR(1).
- Builds LL(1) and SLR(1) parsing tables, detecting conflicts.
- Analyzes strings with either parser, providing detailed output.
- Handles epsilon productions and cyclic dependencies.

## Requirements
- Python 3.6 or higher (no external libraries required).

## Installation
1. Clone or download this repository.
2. Ensure Python is installed (check with `python --version`).
3. Place grammar files (`ejemplo1.txt`, `ejemplo2.txt`, `ejemplo3.txt`) in the same directory.

## Usage
1. Prepare a grammar file (e.g., `ejemplo1.txt`):
   - First line: Number of rules (integer).
   - Following lines: Rules in `NonTerminal -> Derivation` format (e.g., `S -> S + T` or `F -> e` for epsilon).
   - Input strings follow, ending with `e`.
   - Example:
5
S -> S + T
S -> T
T -> F
F -> ( S )
F -> 1
1+1
(1)
(1+1)*1
e



## Limitations
- Requires strict input format.
- Assumes uppercase for non-terminals, lowercase/symbols for terminals.
- May fail with complex grammars having unresolved conflicts.

## Contributing
Fork this repository and submit pull requests. Report issues via GitHub Issues.

## Contact
For questions, contact [your-email@example.com] or open an issue.

## License
MIT License.
"""

from collections import defaultdict  # Importamos defaultdict para crear diccionarios con valores por defecto

# Variables globales
reglas_gramatica = []  # Lista que almacenará las reglas de la gramática (ejemplo: [['S', 'S+T'], ['S', 'T']])
cadenas_a_procesar = []  # Lista que almacenará las cadenas a analizar (ejemplo: ['1+1$', '(1)$'])
simbolo_inicial = None  # Símbolo inicial de la gramática (ejemplo: 'S')

# Función para leer la gramática desde un archivo
def leer_gramatica(nombre_archivo):
    """
    Lee la gramática y las cadenas a procesar desde un archivo de texto.
    Args:
        nombre_archivo (str): Nombre del archivo (ejemplo: "ejemplo1.txt").
    Raises:
        FileNotFoundError: Si el archivo no se encuentra.
        ValueError: Si la primera línea no es un número entero.
    """
    global reglas_gramatica, cadenas_a_procesar, simbolo_inicial  # Usamos variables globales para modificarlas
    reglas_gramatica = []  # Reiniciamos la lista de reglas para empezar de cero
    cadenas_a_procesar = []  # Reiniciamos la lista de cadenas para empezar de cero
    try:
        with open(nombre_archivo, "r") as archivo:  # Abrimos el archivo en modo lectura
            numero_reglas = int(archivo.readline().strip())  # Leemos la primera línea y la convertimos a entero (número de reglas)
            print(f"\nProcesando {nombre_archivo}: {numero_reglas} reglas.")  # Mostramos cuántas reglas se procesarán
            for i in range(numero_reglas):  # Iteramos para leer cada regla
                regla = archivo.readline().strip()  # Leemos una línea del archivo y quitamos espacios
                partes = regla.split("->")  # Dividimos la línea en el no terminal y sus derivaciones usando "->"
                if len(partes) == 2:  # Verificamos que la línea tenga el formato correcto (no_terminal -> derivaciones)
                    partes = [parte.strip() for parte in partes]  # Quitamos espacios a cada parte
                else:
                    print("Formato incorrecto en el archivo.")  # Si el formato es incorrecto, mostramos error y salimos
                    exit()
                derivaciones = partes[1].split(" ")  # Dividimos las derivaciones en una lista (ejemplo: "S+T" -> ['S', '+', 'T'])
                if i == 0:  # Si es la primera regla, tomamos su no terminal como símbolo inicial
                    simbolo_inicial = partes[0]  # Guardamos el símbolo inicial (ejemplo: 'S')
                    print(f"Símbolo inicial: {simbolo_inicial}")  # Mostramos el símbolo inicial
                for derivacion in derivaciones:  # Iteramos sobre las derivaciones de la regla
                    if derivacion == 'e':  # Si la derivación es epsilon
                        derivacion = ''  # La tratamos como una cadena vacía para que tenga longitud 0
                    reglas_gramatica.append([partes[0], derivacion])  # Agregamos cada regla a la lista (ejemplo: ['S', 'S+T'])
            for linea in archivo:  # Leemos el resto del archivo para las cadenas a procesar
                linea = linea.strip() + "$"  # Quitamos espacios y añadimos '$' al final de cada cadena
                if linea == "e$":  # Si encontramos "e$", terminamos de leer cadenas
                    break
                cadenas_a_procesar.append(linea)  # Agregamos la cadena a la lista (ejemplo: '1+1$')
    except FileNotFoundError:
        print(f"Archivo {nombre_archivo} no encontrado.")  # Si el archivo no existe, mostramos error y salimos
        exit()
    except ValueError:
        print("La primera línea debe ser un número entero.")  # Si la primera línea no es un número, mostramos error y salimos
        exit()

# Identificar no terminales y terminales
def identificar_simbolos():
    """
    Identifica los no terminales y terminales de la gramática.
    Returns:
        tuple: (conjunto_no_terminales, conjunto_terminales).
    """
    conjunto_no_terminales = set()  # Creamos un conjunto vacío para los no terminales (letras mayúsculas)
    conjunto_terminales = set()  # Creamos un conjunto vacío para los terminales (letras minúsculas y símbolos)
    for no_terminal, derivacion in reglas_gramatica:  # Iteramos sobre cada regla de la gramática
        conjunto_no_terminales.add(no_terminal)  # Agregamos el no terminal al conjunto (ejemplo: 'S')
        for simbolo in derivacion:  # Iteramos sobre cada símbolo de la derivación
            if simbolo.isupper():  # Si el símbolo es una letra mayúscula, es un no terminal
                conjunto_no_terminales.add(simbolo)  # Lo agregamos al conjunto de no terminales
            elif simbolo != 'e':  # Si no es 'e' (epsilon), es un terminal
                conjunto_terminales.add(simbolo)  # Lo agregamos al conjunto de terminales (ejemplo: '+', '1')
    conjunto_terminales.add('$')  # Agregamos el símbolo de fin de cadena '$' a los terminales
    return conjunto_no_terminales, conjunto_terminales  # Retornamos ambos conjuntos

# Calcular conjuntos First
def calcular_conjuntos_primeros(conjunto_no_terminales, conjunto_terminales):
    """
    Calcula los conjuntos First para cada no terminal.
    Args:
        conjunto_no_terminales (set): Conjunto de no terminales.
        conjunto_terminales (set): Conjunto de terminales.
    Returns:
        dict: Diccionario con los conjuntos First.
    """
    primeros = {nt: set() for nt in conjunto_no_terminales}  # Creamos un diccionario para los conjuntos First, inicialmente vacíos
    hubo_cambios = True  # Bandera para controlar si hay cambios en los conjuntos First
    while hubo_cambios:  # Seguimos iterando mientras haya cambios
        hubo_cambios = False  # Reiniciamos la bandera
        for no_terminal, derivacion in reglas_gramatica:  # Iteramos sobre cada regla de la gramática
            tamano_anterior = len(primeros[no_terminal])  # Guardamos el tamaño actual del conjunto First del no terminal
            if derivacion == '':  # Si la derivación es epsilon (cadena vacía)
                if 'e' not in primeros[no_terminal]:  # Si epsilon no está en el conjunto First
                    primeros[no_terminal].add('e')  # Agregamos epsilon al conjunto First
                    hubo_cambios = True  # Indicamos que hubo un cambio
            else:
                i = 0  # Índice para recorrer los símbolos de la derivación
                todos_tienen_epsilon = True  # Bandera para verificar si todos los símbolos tienen epsilon en su First
                while i < len(derivacion):  # Recorremos los símbolos de la derivación
                    simbolo = derivacion[i]  # Tomamos el símbolo actual
                    if simbolo in conjunto_terminales:  # Si el símbolo es un terminal
                        if simbolo not in primeros[no_terminal]:  # Si el terminal no está en el conjunto First
                            primeros[no_terminal].add(simbolo)  # Lo agregamos
                            hubo_cambios = True  # Indicamos que hubo un cambio
                        todos_tienen_epsilon = False  # Como encontramos un terminal, no todos tienen epsilon
                        break  # Salimos del bucle
                    elif simbolo in conjunto_no_terminales:  # Si el símbolo es un no terminal
                        primeros[no_terminal].update(primeros[simbolo] - {'e'})  # Agregamos el First del no terminal, sin epsilon
                        if 'e' not in primeros[simbolo]:  # Si el First del no terminal no tiene epsilon
                            todos_tienen_epsilon = False  # No todos tienen epsilon
                            break  # Salimos del bucle
                        i += 1  # Avanzamos al siguiente símbolo
                if todos_tienen_epsilon and 'e' not in primeros[no_terminal]:  # Si todos tienen epsilon y no lo hemos añadido
                    primeros[no_terminal].add('e')  # Agregamos epsilon al conjunto First
                    hubo_cambios = True  # Indicamos que hubo un cambio
            if len(primeros[no_terminal]) > tamano_anterior:  # Si el tamaño del conjunto First cambió
                hubo_cambios = True  # Indicamos que hubo un cambio
    return primeros  # Retornamos los conjuntos First calculados

# Calcular conjuntos Follow
def calcular_conjuntos_siguientes(conjunto_no_terminales, conjunto_terminales, primeros):
    """
    Calcula los conjuntos Follow para cada no terminal.
    Args:
        conjunto_no_terminales (set): Conjunto de no terminales.
        conjunto_terminales (set): Conjunto de terminales.
        primeros (dict): Conjuntos First.
    Returns:
        dict: Diccionario con los conjuntos Follow.
    """
    siguientes = {nt: set() for nt in conjunto_no_terminales}  # Creamos un diccionario para los conjuntos Follow, inicialmente vacíos
    if simbolo_inicial in siguientes:  # Si el símbolo inicial está en el diccionario
        siguientes[simbolo_inicial].add('$')  # Agregamos '$' al Follow del símbolo inicial
    visitados = {nt: False for nt in conjunto_no_terminales}  # Diccionario para rastrear no terminales visitados y evitar ciclos
    def agregar_follow(simbolo, no_terminal):  # Función auxiliar para agregar Follow evitando ciclos
        if visitados[simbolo]:  # Si ya visitamos este no terminal
            return  # Evitamos ciclos
        visitados[simbolo] = True  # Marcamos el no terminal como visitado
        tamano_antes = len(siguientes[simbolo])  # Guardamos el tamaño actual del Follow
        siguientes[simbolo].update(siguientes[no_terminal])  # Agregamos el Follow del no terminal
        if len(siguientes[simbolo]) > tamano_antes:  # Si el tamaño cambió
            return True  # Indicamos que hubo un cambio
        return False  # No hubo cambios
    hubo_cambios = True  # Bandera para controlar si hay cambios en los conjuntos Follow
    while hubo_cambios:  # Seguimos iterando mientras haya cambios
        hubo_cambios = False  # Reiniciamos la bandera
        visitados = {nt: False for nt in conjunto_no_terminales}  # Reiniciamos los visitados en cada iteración
        for no_terminal, derivacion in reglas_gramatica:  # Iteramos sobre cada regla de la gramática
            for i in range(len(derivacion)):  # Recorremos los símbolos de la derivación
                simbolo = derivacion[i]  # Tomamos el símbolo actual
                if simbolo in conjunto_no_terminales:  # Si el símbolo es un no terminal
                    if i + 1 < len(derivacion):  # Si hay un símbolo siguiente en la derivación
                        siguiente = derivacion[i + 1]  # Tomamos el símbolo siguiente
                        if siguiente in conjunto_terminales:  # Si el siguiente es un terminal
                            tamano_antes = len(siguientes[simbolo])  # Guardamos el tamaño actual del Follow
                            siguientes[simbolo].add(siguiente)  # Agregamos el terminal al Follow
                            if len(siguientes[simbolo]) > tamano_antes:  # Si el tamaño cambió
                                hubo_cambios = True  # Indicamos que hubo un cambio
                        elif siguiente in conjunto_no_terminales:  # Si el siguiente es un no terminal
                            tamano_antes = len(siguientes[simbolo])  # Guardamos el tamaño actual del Follow
                            siguientes[simbolo].update(primeros[siguiente] - {'e'})  # Agregamos el First del siguiente, sin epsilon
                            if len(siguientes[simbolo]) > tamano_antes:  # Si el tamaño cambió
                                hubo_cambios = True  # Indicamos que hubo un cambio
                            if 'e' in primeros[siguiente] and no_terminal != simbolo:  # Si el First del siguiente tiene epsilon
                                if agregar_follow(simbolo, no_terminal):  # Agregamos el Follow del no terminal
                                    hubo_cambios = True  # Indicamos que hubo un cambio
                    else:  # Si no hay símbolo siguiente (estamos al final de la derivación)
                        if no_terminal != simbolo:  # Si el no terminal no es el mismo símbolo
                            if agregar_follow(simbolo, no_terminal):  # Agregamos el Follow del no terminal
                                hubo_cambios = True  # Indicamos que hubo un cambio
    return siguientes  # Retornamos los conjuntos Follow calculados

# Construir tabla LL(1)
def construir_tabla_ll1(conjunto_no_terminales, conjunto_terminales, primeros, siguientes):
    """
    Construye la tabla de análisis LL(1).
    Args:
        conjunto_no_terminales (set): Conjunto de no terminales.
        conjunto_terminales (set): Conjunto de terminales.
        primeros (dict): Conjuntos First.
        siguientes (dict): Conjuntos Follow.
    Returns:
        tuple: (tabla_ll1, conflictos).
    """
    tabla_ll1 = {nt: {t: None for t in conjunto_terminales} for nt in conjunto_no_terminales}  # Creamos la tabla LL(1), inicialmente vacía
    conflictos = False  # Bandera para detectar conflictos en la tabla
    for no_terminal, derivacion in reglas_gramatica:  # Iteramos sobre cada regla de la gramática
        primeros_derivacion = set()  # Creamos un conjunto para los First de la derivación
        i = 0  # Índice para recorrer los símbolos de la derivación
        todos_tienen_epsilon = True  # Bandera para verificar si todos los símbolos tienen epsilon
        while i < len(derivacion):  # Recorremos los símbolos de la derivación
            simbolo = derivacion[i]  # Tomamos el símbolo actual
            if simbolo in conjunto_terminales:  # Si el símbolo es un terminal
                primeros_derivacion.add(simbolo)  # Agregamos el terminal al conjunto First de la derivación
                todos_tienen_epsilon = False  # Como encontramos un terminal, no todos tienen epsilon
                break  # Salimos del bucle
            elif simbolo in conjunto_no_terminales:  # Si el símbolo es un no terminal
                primeros_derivacion.update(primeros[simbolo] - {'e'})  # Agregamos el First del no terminal, sin epsilon
                if 'e' not in primeros[simbolo]:  # Si el First del no terminal no tiene epsilon
                    todos_tienen_epsilon = False  # No todos tienen epsilon
                    break  # Salimos del bucle
                i += 1  # Avanzamos al siguiente símbolo
        if todos_tienen_epsilon:  # Si todos los símbolos tienen epsilon
            primeros_derivacion.add('e')  # Agregamos epsilon al First de la derivación
        for terminal in primeros_derivacion - {'e'}:  # Para cada terminal en el First, sin epsilon
            if tabla_ll1[no_terminal][terminal] is not None:  # Si ya hay una entrada en la tabla
                print(f"Conflicto en [{no_terminal}, {terminal}]: {tabla_ll1[no_terminal][terminal]} vs {derivacion}")  # Mostramos conflicto
                conflictos = True  # Indicamos que hay un conflicto
            tabla_ll1[no_terminal][terminal] = derivacion  # Asignamos la derivación a la tabla
        if 'e' in primeros_derivacion:  # Si epsilon está en el First de la derivación
            for terminal in siguientes[no_terminal]:  # Para cada terminal en el Follow del no terminal
                if tabla_ll1[no_terminal][terminal] is not None:  # Si ya hay una entrada en la tabla
                    print(f"Conflicto en [{no_terminal}, {terminal}]: {tabla_ll1[no_terminal][terminal]} vs {derivacion}")  # Mostramos conflicto
                    conflictos = True  # Indicamos que hay un conflicto
                tabla_ll1[no_terminal][terminal] = derivacion  # Asignamos la derivación a la tabla
    return tabla_ll1, conflictos  # Retornamos la tabla y si hubo conflictos

# Analizar con LL(1)
def analizar_ll1(cadena, tabla_ll1, conjunto_no_terminales, conjunto_terminales):
    """
    Analiza una cadena usando el analizador LL(1).
    Args:
        cadena (str): Cadena de entrada.
        tabla_ll1 (dict): Tabla de análisis LL(1).
        conjunto_no_terminales (set): Conjunto de no terminales.
        conjunto_terminales (set): Conjunto de terminales.
    Returns:
        bool: True si la cadena es aceptada, False si no.
    """
    pila = [simbolo_inicial, '$']  # Inicializamos la pila con el símbolo inicial y '$'
    i = 0  # Índice para recorrer la cadena de entrada
    print(f"\nAnalizando {cadena} con LL(1)")  # Mostramos la cadena que vamos a analizar
    while pila:  # Mientras la pila no esté vacía
        tope = pila[-1]  # Tomamos el elemento en la cima de la pila
        print(f"Pila: {pila}, Entrada: {cadena[i:]}")  # Mostramos el estado actual de la pila y la entrada
        if tope in conjunto_terminales:  # Si el tope de la pila es un terminal
            if i >= len(cadena):  # Si ya no hay más entrada
                print("Error: Entrada agotada.")  # Mostramos error
                return False  # La cadena no es aceptada
            if tope == cadena[i]:  # Si el terminal coincide con el símbolo de la entrada
                pila.pop()  # Sacamos el terminal de la pila
                i += 1  # Avanzamos en la entrada
            else:
                print(f"Error: {tope} no coincide con {cadena[i]}")  # Si no coincide, mostramos error
                return False  # La cadena no es aceptada
        elif tope == '$':  # Si el tope de la pila es '$'
            if i == len(cadena):  # Si hemos procesado toda la entrada
                print("Cadena aceptada.")  # Mostramos que la cadena es aceptada
                return True  # Retornamos True
            print("Error: Pila vacía con entrada restante.")  # Si queda entrada, mostramos error
            return False  # La cadena no es aceptada
        else:  # Si el tope de la pila es un no terminal
            if i >= len(cadena):  # Si ya no hay más entrada
                print("Error: Entrada agotada con no terminales.")  # Mostramos error
                return False  # La cadena no es aceptada
            simbolo_entrada = cadena[i]  # Tomamos el símbolo actual de la entrada
            if simbolo_entrada not in conjunto_terminales:  # Si el símbolo no es un terminal
                print(f"Error: {simbolo_entrada} no es terminal.")  # Mostramos error
                return False  # La cadena no es aceptada
            if tabla_ll1[tope][simbolo_entrada] is None:  # Si no hay regla en la tabla para este caso
                print(f"Error: No hay regla para [{tope}, {simbolo_entrada}]")  # Mostramos error
                return False  # La cadena no es aceptada
            produccion = tabla_ll1[tope][simbolo_entrada]  # Obtenemos la producción de la tabla
            pila.pop()  # Sacamos el no terminal de la pila
            if produccion != '':  # Si la producción no es epsilon
                for simbolo in reversed(produccion):  # Recorremos los símbolos de la producción en orden inverso
                    pila.append(simbolo)  # Los agregamos a la pila
    return False  # Si salimos del bucle sin aceptar, retornamos False

# Construcción de ítems LR(0) para SLR(1)
def cerrar_item(item, conjunto_no_terminales):
    """
    Calcula el cierre de un ítem LR(0).
    Args:
        item (tuple): Ítem en formato ((no_terminal, derivacion), posicion).
        conjunto_no_terminales (set): Conjunto de no terminales.
    Returns:
        frozenset: Conjunto de ítems cerrados (usamos frozenset para que sea hasheable).
    """
    items_cerrados = {item}  # Inicializamos el conjunto de ítems cerrados con el ítem inicial
    hubo_cambios = True  # Bandera para controlar si hay cambios en el cierre
    while hubo_cambios:  # Seguimos iterando mientras haya cambios
        hubo_cambios = False  # Reiniciamos la bandera
        tamano_actual = len(items_cerrados)  # Guardamos el tamaño actual del conjunto de ítems
        nuevos_items = set()  # Creamos un conjunto para los nuevos ítems que vamos a agregar
        for (no_terminal, derivacion), posicion in items_cerrados:  # Iteramos sobre cada ítem en el cierre
            if posicion < len(derivacion):  # Si el punto no está al final de la derivación
                if derivacion[posicion].isupper():  # Si el símbolo después del punto es un no terminal
                    nt = derivacion[posicion]  # Tomamos el no terminal (ejemplo: 'S')
                    for no_term, deriv in reglas_gramatica:  # Iteramos sobre todas las reglas de la gramática
                        if no_term == nt:  # Si el no terminal de la regla coincide
                            nuevo_item = ((no_term, deriv), 0)  # Creamos un nuevo ítem con el punto al inicio
                            if nuevo_item not in items_cerrados:  # Si el ítem no está ya en el cierre
                                nuevos_items.add(nuevo_item)  # Lo agregamos a los nuevos ítems
        items_cerrados.update(nuevos_items)  # Agregamos los nuevos ítems al cierre
        if len(items_cerrados) > tamano_actual:  # Si el tamaño del cierre cambió
            hubo_cambios = True  # Indicamos que hubo un cambio
    return frozenset(items_cerrados)  # Retornamos el cierre como un frozenset para que sea hasheable

def ir_a(estado, simbolo, conjunto_no_terminales):
    """
    Calcula el estado siguiente (goto).
    Args:
        estado (frozenset): Conjunto de ítems.
        simbolo (str): Símbolo para la transición.
        conjunto_no_terminales (set): Conjunto de no terminales.
    Returns:
        frozenset: Nuevo estado.
    """
    nuevo_estado = set()  # Creamos un conjunto vacío para el nuevo estado
    for (no_terminal, derivacion), posicion in estado:  # Iteramos sobre cada ítem en el estado
        if posicion < len(derivacion):  # Si el punto no está al final de la derivación
            if derivacion[posicion] == simbolo:  # Si el símbolo después del punto coincide con el símbolo dado
                nuevo_estado.add(((no_terminal, derivacion), posicion + 1))  # Avanzamos el punto y agregamos el ítem
    if nuevo_estado:  # Si encontramos ítems para el nuevo estado
        return cerrar_item(frozenset(nuevo_estado), conjunto_no_terminales)  # Calculamos el cierre del nuevo estado
    return frozenset()  # Si no hay ítems, retornamos un frozenset vacío

# Verificar si es SLR(1)
def es_slr1(conjunto_no_terminales, conjunto_terminales, siguientes):
    """
    Verifica si la gramática es SLR(1).
    Args:
        conjunto_no_terminales (set): Conjunto de no terminales.
        conjunto_terminales (set): Conjunto de terminales.
        siguientes (dict): Conjuntos Follow.
    Returns:
        bool: True si es SLR(1), False si no.
    """
    estados = []  # Lista para almacenar todos los estados LR(0)
    item_inicial = cerrar_item(((simbolo_inicial + "'", simbolo_inicial), 0), conjunto_no_terminales)  # Calculamos el cierre del ítem inicial
    estados.append(item_inicial)  # Agregamos el estado inicial a la lista de estados

    i = 0  # Índice para recorrer los estados
    while i < len(estados):  # Mientras haya estados por procesar
        estado = estados[i]  # Tomamos el estado actual
        simbolos = set()  # Conjunto para almacenar los símbolos después del punto
        for (no_terminal, derivacion), posicion in estado:  # Iteramos sobre cada ítem en el estado
            if posicion < len(derivacion):  # Si el punto no está al final de la derivación
                simbolos.add(derivacion[posicion])  # Agregamos el símbolo después del punto
        for simbolo in simbolos:  # Para cada símbolo encontrado
            siguiente_estado = ir_a(estado, simbolo, conjunto_no_terminales)  # Calculamos el estado siguiente (goto)
            if siguiente_estado and siguiente_estado not in estados:  # Si el estado no está vacío y no lo hemos visto
                estados.append(siguiente_estado)  # Lo agregamos a la lista de estados
        i += 1  # Avanzamos al siguiente estado

    conflictos = False  # Bandera para detectar conflictos
    for idx_estado, estado in enumerate(estados):  # Iteramos sobre cada estado con su índice
        reducciones = []  # Lista para almacenar las reducciones posibles en este estado
        desplazamientos = set()  # Conjunto para almacenar los símbolos de desplazamiento
        for (no_terminal, derivacion), posicion in estado:  # Iteramos sobre cada ítem en el estado
            if posicion == len(derivacion):  # Si el punto está al final (reducción)
                if no_terminal == simbolo_inicial + "'":  # Si es la regla inicial aumentada
                    continue  # La ignoramos (se manejará como aceptación)
                reducciones.append((no_terminal, derivacion))  # Agregamos la reducción a la lista
            elif posicion < len(derivacion):  # Si el punto no está al final (desplazamiento)
                simbolo = derivacion[posicion]  # Tomamos el símbolo después del punto
                if simbolo in conjunto_terminales:  # Si el símbolo es un terminal
                    desplazamientos.add(simbolo)  # Agregamos el símbolo al conjunto de desplazamientos
        for no_terminal, _ in reducciones:  # Para cada reducción
            for t in siguientes[no_terminal]:  # Para cada terminal en el Follow del no terminal
                if t in desplazamientos:  # Si el terminal también permite un desplazamiento
                    print(f"Conflicto shift-reduce en estado {idx_estado} con {t}")  # Mostramos conflicto shift-reduce
                    conflictos = True  # Indicamos que hay un conflicto
        if len(reducciones) > 1:  # Si hay más de una reducción posible
            print(f"Conflicto reduce-reduce en estado {idx_estado}")  # Mostramos conflicto reduce-reduce
            conflictos = True  # Indicamos que hay un conflicto
    return not conflictos  # Retornamos True si no hay conflictos (es SLR(1))

# Construir
