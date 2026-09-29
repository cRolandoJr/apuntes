# El Lenguaje Nix desde cero

## Por qué existe un lenguaje propio

YAML y JSON son datos estáticos — no hay variables, condicionales, ni reutilización. Python sería demasiado poderoso y con efectos secundarios.

Nix el lenguaje está diseñado con restricciones específicas:

- **Funcional puro:** una expresión siempre produce el mismo resultado. Sin estado global, sin efectos secundarios
- **Lazy (evaluación perezosa):** solo evalúa lo que necesita. nixpkgs tiene miles de paquetes, pero si instalás solo `firefox`, Nix no evalúa nada que no uses
- **Orientado a datos:** todo termina siendo un valor

---

## Todo en Nix es una expresión

No hay sentencias, no hay `print()`, no hay `return`. **Todo es una expresión que produce un valor.**

```nix
40 + 2        # → 42
"hola"        # → "hola"
3 > 2         # → true
```

---

## Tipos de datos

### Números y booleanos

```nix
42            # entero
-7
3.14          # flotante (se usa poco)
true
false
null
```

### Strings

```nix
# String normal
"hola mundo"

# Interpolación con ${}
let nombre = "Rolando";
in "Hola, ${nombre}!"
# → "Hola, Rolando!"

# String multilínea — ignora la indentación común
''
  #!/bin/bash
  echo "hola"
  exit 0
''

# Path — rutas de archivo (tipo propio, NO string)
/etc/nixos/configuration.nix
./mi-archivo.nix    # relativa al archivo .nix actual
```

> **Importante:** los paths sin comillas al evaluarse se copian a la Nix Store. Con comillas son solo texto.

### Listas

```nix
# Elementos separados por ESPACIOS, no comas
[ 1 2 3 ]
[ "firefox" "neovim" "git" ]
[ ]    # vacía

# Concatenar listas
[ 1 2 ] ++ [ 3 4 ]    # → [ 1 2 3 4 ]
```

### Attribute Sets — el tipo más importante

Como un diccionario o un objeto JSON. Es la estructura central de Nix.

```nix
# Sintaxis básica
{
  nombre = "Rolando";
  edad = 26;
  activo = true;
}

# Acceder con punto
let persona = { nombre = "Rolando"; edad = 26; };
in persona.nombre
# → "Rolando"

# Atributos anidados — dos formas equivalentes
{
  servicios = {
    ssh = {
      enable = true;
      port = 22;
    };
  };
}

# Forma abreviada con punto (más común en NixOS)
{
  servicios.ssh.enable = true;
  servicios.ssh.port = 22;
}

# Merge de dos sets (el derecho gana en conflicto)
{ a = 1; b = 2; } // { b = 99; c = 3; }
# → { a = 1; b = 99; c = 3; }
```

Cuando en `configuration.nix` ves:
```nix
services.openssh.enable = true;
```
Es exactamente un attribute set anidado.

---

## `let ... in` — variables locales

```nix
let
  x = 10;
  y = 20;
  suma = x + y;
in
  suma * 2
# → 60
```

El `let` define nombres. El `in` es la expresión que se evalúa usando esos nombres.

```nix
# Podés referenciar bindings anteriores
let
  base = 1920;
  altura = 1080;
  total = base * altura;
in
  "Resolución: ${toString base}x${toString altura} = ${toString total} px"
```

---

## `with` — abrir un attribute set

```nix
# Sin with — repetitivo
[ pkgs.firefox pkgs.neovim pkgs.git pkgs.htop ]

# Con with — más limpio
with pkgs; [ firefox neovim git htop ]
```

> **Advertencia:** `with` puede hacer el código difícil de leer en configuraciones grandes porque no es obvio de dónde viene cada nombre. En listas de paquetes es el uso más aceptado.

---

## Funciones — la parte más importante

### Sintaxis básica

```nix
# argumento: cuerpo
x: x * 2

# Llamar — sin paréntesis, sin comas
(x: x * 2) 10
# → 20
```

Las funciones en Nix toman **exactamente un argumento**. Para múltiples, se encadenan (currying):

```nix
a: b: a + b

(a: b: a + b) 3 5
# → 8  (equivale a: ((a: b: a + b) 3) 5)
```

### Funciones con attribute set — la más común

```nix
# Argumentos nombrados
{ nombre, edad }: "Hola ${nombre}, tenés ${toString edad} años"

# Con valor por defecto
{ nombre, edad ? 0 }: "Hola ${nombre}, tenés ${toString edad} años"

# Con ... para permitir atributos extras
{ nombre, edad, ... }: nombre

# Con @ para capturar el set completo
{ nombre, edad, ... }@args: args
```

Esto es lo que ves al principio de cualquier módulo NixOS:

```nix
{ config, pkgs, lib, ... }:   # ← función con argumento set
{
  environment.systemPackages = with pkgs; [ firefox ];
}
```

El archivo entero *es* una función. NixOS la llama pasándole el contexto del sistema.

---

## `inherit` — evitar repetición

```nix
let
  nombre = "Rolando";
  edad = 26;
in {
  # Sin inherit
  nombre = nombre;
  edad = edad;

  # Con inherit — equivalente
  inherit nombre edad;
}

# Heredar de otro set
let persona = { nombre = "Rolando"; ciudad = "Viedma"; };
in {
  inherit (persona) nombre ciudad;
  pais = "Argentina";
}
```

---

## `import` — separar código en archivos

```nix
# Si hardware.nix contiene: { cpuFreq = 3200; }
let hardware = import ./hardware.nix;
in hardware.cpuFreq
# → 3200

# Si el archivo es una función, pasarle argumentos
import ./modulo.nix { pkgs = ...; }
```

Lo que hace NixOS con tus módulos:

```nix
imports = [
  ./hardware-configuration.nix
  ./home.nix
];
```

Importa, evalúa las funciones pasándoles el contexto, y fusiona todos los attribute sets resultantes.

---

## `if-then-else` — condicional (también es expresión)

```nix
if 3 > 2 then "mayor" else "menor"
# → "mayor"

# En configuración real
services.openssh.enable = if config.networking.hostName == "laptop" then true else false;
```

---

## `builtins` y `lib`

```nix
# builtins — siempre disponibles
builtins.toString 42                        # → "42"
builtins.length [ 1 2 3 ]                   # → 3
builtins.map (x: x * 2) [ 1 2 3 ]          # → [ 2 4 6 ]
builtins.filter (x: x > 2) [ 1 2 3 4 ]     # → [ 3 4 ]
builtins.readFile ./archivo.txt

# lib — disponible en módulos con nixpkgs
lib.strings.toUpper "hola"                  # → "HOLA"
lib.lists.flatten [ [ 1 2 ] [ 3 4 ] ]       # → [ 1 2 3 4 ]
lib.optionals condicion [ pkgs.git ]        # lista vacía si condicion es false
```

---

## Ejemplo real — módulo NixOS completo

```nix
# Este archivo ES una función
{ config, pkgs, lib, ... }:

let
  usuario = "rolando";
  paquetesDesarrollo = with pkgs; [
    go
    python3
    nodejs
    git
  ];
in
{
  # Attribute set que NixOS fusiona con el resto de la configuración

  users.users.${usuario} = {        # interpolación en clave de set
    isNormalUser = true;
    extraGroups = [ "wheel" "docker" ];
  };

  environment.systemPackages = paquetesDesarrollo ++ (with pkgs; [
    firefox
    neovim
  ]);

  services.openssh.enable = true;
}
```

---

## Referencia rápida

| Concepto | Sintaxis | Para qué |
|----------|----------|----------|
| String | `"texto"` / `''multilínea''` | Texto con interpolación `${}` |
| Path | `/ruta` / `./relativa` | Rutas de archivo |
| Lista | `[ a b c ]` | Colección ordenada |
| Attribute set | `{ clave = valor; }` | Diccionario / objeto |
| Variable local | `let x = 1; in x` | Nombres dentro de un scope |
| Función | `x: expresión` | Transformación de valores |
| Función con set | `{ a, b }: expresión` | Argumentos nombrados |
| Importar archivo | `import ./archivo.nix` | Modularizar configuración |
| Abrir un set | `with pkgs; [ firefox ]` | Evitar prefijos repetitivos |
| Evitar repetición | `inherit nombre;` | Shorthand para claves iguales |
| Concatenar listas | `lista1 ++ lista2` | Juntar listas |
| Merge de sets | `set1 // set2` | Fusionar (derecho gana) |
| Condicional | `if cond then a else b` | Lógica condicional |
