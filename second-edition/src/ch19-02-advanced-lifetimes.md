## Vida útil avanzada

De vuelta en el Capítulo 10 en la sección "Validación de referencias con tiempos de vida",
Aprendí a anotar referencias con parámetros de por vida para decirle a Rust cómo
vidas de diferentes referencias se relacionan. Vimos cómo cada referencia puede durar
de por vida, pero, la mayoría de las veces, Rust te permitirá olvidarlas. Aquí vamos a
mira tres características avanzadas de vidas que aún no hemos cubierto:


* Subtipo de por vida, una forma de asegurar que una vida sobreviva a otra para siempre.
* Límites de por vida, para especificar un tiempo de vida para una referencia a un tipo genérico.
* Duración de vida de los objetos, cómo se infieren y cuándo deben ser  especificados


<!-- ¿debería agregar un pequeño resumen de cada uno aquí? 
Eso nos permitiría aclarar directamente en ejemplos en la 
siguiente sección -> <! - He cambiado a viñetas y agregué un 
pequeño resumen / Carol -->

### La subtipificación de por vida garantiza que una vida sobrevive a otra

El subtipado de por vida es una forma de especificar que una vida 
debe sobrevivir a otra de por vida. Para explorar la subtipificación 
de por vida, imagina que queremos escribir un analizador sintáctico. 
Tendremos una estructura llamada `Context` que contiene una referencia
a la cadena que estamos analizando, Escribiremos un analizador que 
analizará esta cadena y devolverá éxito o fracaso. El analizador 
necesitará tomar prestado el contexto para hacer el analizando. 
Implementar esto se vería como el código en el listado 19-12, excepto
que este código no tiene las anotaciones de vida requeridas para que no se compile:

<span class="filename">Nombre del archivo: src/lib.rs</span>

```rust,ignore
struct Context(&str);

struct Parser {
    context: &Context,
}

impl Parser {
    fn parse(&self) -> Result<(), &str> {
        Err(&self.context.0[1..])
    }
}
```

<span class="caption">Listing 19-12: Definiendo anotacines de un analizador 
  sin tiempo de vida</span>

La compilación del código da como resultado errores que indican que 
Rust espera una vida útil con parámetros en el segmento de cadena en  
`Context` y la referencia a `Context` en `Parser`.

<!-- ¿Cuál será el error de tiempo de compilación aquí? Creo que valdría la pena mostrar
eso para el lector ->
<! - Los errores solo dicen "parámetro de por vida esperado", son bastante aburridos.
Hemos mostrado mensajes de error como ese anteriormente, así que lo he explicado en palabras.
/ Carol -->


En aras de la simplicidad, nuestra función `parse` devuelve un
`Result <(), & str> `. Este, no hará nada en el éxito, y en el 
fracaso devolverá la parte de la segmento de cadena que no se 
analizó correctamente. Una implementación real tendría más información
de error que eso, y realmente devolvería algo al analizar si tiene éxito,
pero los dejaremos porque no son relevantes para el vidas que forman parte de este ejemplo.

Para mantener este código simple, no vamos a escribir ninguna lógica de análisis.
Es muy probable que en algún lugar de la lógica de análisis manejemos la entrada no válida por
devolver un error que hace referencia a la parte de la entrada que no es válida, y
esta referencia es lo que hace que el ejemplo de código sea interesante con respecto a las
vidas, Entonces vamos a pretender que la lógica de nuestro analizador es que
la entrada no es válida después del primer byte. Tenga en cuenta que este 
código puede entrar en pánico si el primer byte no está en un límite de 
caracteres válidos; de nuevo, estamos simplificando
ejemplo para concentrarse en las vidas involucradas.


<!-- ¿Por qué queremos siempre error después del primer byte? ->
<! - Por simplicidad del ejemplo para evitar saturar el código 
real Analizando la lógica, que no es el punto. He explicado un poco más arriba / Carol ->

Para obtener este código compilando, necesitamos completar los
parámetros de vida para el string slice en `Context` y la referencia
al` Context` en `Parser`. La forma más sencilla de hacer esto es usar
la misma vida en todas partes, como se muestra en el listado 19-13:

<span class="filename">Nombre del archivo: src/lib.rs</span>

```rust
struct Context<'a>(&'a str);

struct Parser<'a> {
    context: &'a Context<'a>,
}

impl<'a> Parser<'a> {
    fn parse(&self) -> Result<(), &str> {
        Err(&self.context.0[1..])
    }
}
```

<span class="caption">Listado 19-13: Anotando todas las referencias en `Context` 
  y `Parser` con el mismo parámetro de vida</span>

Esto compila bien y le dice a Rust que un `Parser` contiene una referencia 
a un `Context` con lifetime `a`, y que `Context` contiene un segmento de 
cadena que también vive tanto tiempo como la referencia al `Context` en `Parser` 
.El compilador de Rust envía un mensaje de error con dichos parámetros de por
vida que fueron necesarios para estas referencias, y ahora hemos agregado parámetros de por vida.

<!-- ¿Puede dejar que el lector sepa que deberían estar tomando distancia 
de este anterior ¿ejemplo? No tengo muy claro por qué agregar vidas aquí 
guardado el código -> <! - Done -->

Luego, en el listado 19-14, agreguemos una función que toma una instancia
de `Context`, usa un` Parser` para analizar ese contexto, y devuelve `parse`. 
Esto no funcionará del todo:

<span class="filename">Nombre del archivo: src/lib.rs</span>

```rust,ignore
fn parse_context(context: Context) -> Result<(), &str> {
    Parser { context: &context }.parse()
}
```

<span class="caption">Listado 19-14: Un intento de agregar una funcion `parse_context` que 
  toma un `Contexto` y utiliza un `Parser` </span>

Obtenemos dos errores bastante detallados cuando tratamos de compilar el 
código con el Además de la función `parse_context`:

```text
error[E0597]: borrowed value does not live long enough
  --> src/lib.rs:14:5
   |
14 |     Parser { context: &context }.parse()
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ does not live long enough
15 | }
   | - temporary value only lives until here
   |
note: borrowed value must be valid for the anonymous lifetime #1 defined on the function body at 13:1...
  --> src/lib.rs:13:1
   |
13 | / fn parse_context(context: Context) -> Result<(), &str> {
14 | |     Parser { context: &context }.parse()
15 | | }
   | |_^

error[E0597]: `context` does not live long enough
  --> src/lib.rs:14:24
   |
14 |     Parser { context: &context }.parse()
   |                        ^^^^^^^ does not live long enough
15 | }
   | - borrowed value only lives until here
   |
note: borrowed value must be valid for the anonymous lifetime #1 defined on the function body at 13:1...
  --> src/lib.rs:13:1
   |
13 | / fn parse_context(context: Context) -> Result<(), &str> {
14 | |     Parser { context: &context }.parse()
15 | | }
   | |_^
```

Estos errores dicen que tanto la instancia `Parser` que se creó como
`context` parameter live only desde cuando se crea el` Parser` hasta el final
de la función `parse_context`, necesitan vivir en su totalidad para la
vida de la función.


En otras palabras, `Parser` y` context` necesitan * sobrevivir * a la 
función completa y ser válido antes de que la función comience tan bien 
como después de que termine en orden para que todas las referencias en 
este código siempre sean válidas. Tanto el `Parser` que estamos
creando y el parámetro `context` sale fuera del alcance al final de
función, sin embargo (porque `parse_context` toma posesión de` context`).


<!-- Oh interesante, ¿por qué necesitan sobrevivir a la función?, simplemente
para aseguran absolutamente de que vivirán mientras dure la función --> 
<!-- Sí, es lo que creo que hemos dicho en la primera frase del párrafo anterior.
¿Hay algo que no está claro? / Carol -->

Para descubrir por qué estamos recibiendo estos errores, veamos las 
definiciones en Listado 19-13 nuevamente, específicamente 
las referencias en la firma del Método `parse`:

```rust,ignore
    fn parse(&self) -> Result<(), &str> {
```

<!-- What exactly is it the reader should be looking at in this signature? -->
<!-- Added above /Carol -->

¿Recuerdas las reglas de elision? Si anotamos las vidas de las 
referencias en lugar de elidir, la firma sería:

```rust,ignore
    fn parse<'a>(&'a self) -> Result<(), &'a str> {
```

Es decir, la parte de error del valor de retorno de `parse` tiene una vida que esta
atada a la vida de la instancia `Parser` (la de` & self` en `parse`
firma del método). Eso tiene sentido: el segmento de cadena devuelto hace referencia a los
segmento de cadena en la instancia `Context` sostenida por `Parser`, y la definición
de la estructura `Parser` especifica que la duración de la referencia a
`Context` y el tiempo de vida del segmento de cadena que contiene `Context` deben ser
lo mismo.


The problem is that the `parse_context` function returns the value returned
from `parse`, so the lifetime of the return value of `parse_context` is tied to
the lifetime of the `Parser` as well. But the `Parser` instance created in the
`parse_context` function won’t live past the end of the function (it’s
temporary), and `context` will go out of scope at the end of the function
(`parse_context` takes ownership of it).

Rust thinks we’re trying to return a reference to a value that goes out of
scope at the end of the function, because we annotated all the lifetimes with
the same lifetime parameter. That told Rust the lifetime of the string slice
that `Context` holds is the same as that of the lifetime of the reference to
`Context` that `Parser` holds.

The `parse_context` function can’t see that within the `parse` function, the
string slice returned will outlive both `Context` and `Parser`, and that the
reference `parse_context` returns refers to the string slice, not to `Context`
or `Parser`.

By knowing what the implementation of `parse` does, we know that the only
reason the return value of `parse` is tied to the `Parser` is because it’s
referencing the `Parser`’s `Context`, which is referencing the string slice, so
it’s really the lifetime of the string slice that `parse_context` needs to care
about. We need a way to tell Rust that the string slice in `Context` and the
reference to the `Context` in `Parser` have different lifetimes and that the
return value of `parse_context` is tied to the lifetime of the string slice in
`Context`.

First we’ll try giving `Parser` and `Context` different lifetime parameters as
shown in Listing 19-15. We’ll use `'s` and `'c` as lifetime parameter names to
be clear about which lifetime goes with the string slice in `Context` and which
goes with the reference to `Context` in `Parser`. Note that this won’t
completely fix the problem, but it’s a start and we’ll look at why this isn’t
sufficient when we try to compile.

<span class="filename">Filename: src/lib.rs</span>

```rust,ignore
struct Context<'s>(&'s str);

struct Parser<'c, 's> {
    context: &'c Context<'s>,
}

impl<'c, 's> Parser<'c, 's> {
    fn parse(&self) -> Result<(), &'s str> {
        Err(&self.context.0[1..])
    }
}

fn parse_context(context: Context) -> Result<(), &str> {
    Parser { context: &context }.parse()
}
```

<span class="caption">Listing 19-15: Specifying different lifetime parameters
for the references to the string slice and to `Context`</span>

We’ve annotated the lifetimes of the references in all the same places that we
annotated them in Listing 19-13, but used different parameters depending on
whether the reference goes with the string slice or with `Context`. We’ve also
added an annotation to the string slice part of the return value of `parse` to
indicate that it goes with the lifetime of the string slice in `Context`.

The following is the error we get now when we try to compile:

```text
error[E0491]: in type `&'c Context<'s>`, reference has a longer lifetime than the data it references
 --> src/lib.rs:4:5
  |
4 |     context: &'c Context<'s>,
  |     ^^^^^^^^^^^^^^^^^^^^^^^^
  |
note: the pointer is valid for the lifetime 'c as defined on the struct at 3:1
 --> src/lib.rs:3:1
  |
3 | / struct Parser<'c, 's> {
4 | |     context: &'c Context<'s>,
5 | | }
  | |_^
note: but the referenced data is only valid for the lifetime 's as defined on the struct at 3:1
 --> src/lib.rs:3:1
  |
3 | / struct Parser<'c, 's> {
4 | |     context: &'c Context<'s>,
5 | | }
  | |_^
```

Rust doesn’t know of any relationship between `'c` and `'s`. In order to be
valid, the referenced data in `Context` with lifetime `'s` needs to be
constrained, to guarantee that it lives longer than the reference with lifetime
`'c`. If `'s` is not longer than `'c`, the reference to `Context` might not be
valid.

Which gets us to the point of this section: the Rust feature *lifetime
subtyping* is a way to specify that one lifetime parameter lives at least as
long as another one. In the angle brackets where we declare lifetime
parameters, we can declare a lifetime `'a` as usual, and declare a lifetime
`'b` that lives at least as long as `'a` by declaring `'b` with the syntax `'b:
'a`.

In our definition of `Parser`, in order to say that `'s` (the lifetime of the
string slice) is guaranteed to live at least as long as `'c` (the lifetime of
the reference to `Context`), we change the lifetime declarations to look like
this:

<span class="filename">Filename: src/lib.rs</span>

```rust
# struct Context<'a>(&'a str);
#
struct Parser<'c, 's: 'c> {
    context: &'c Context<'s>,
}
```

Now, the reference to `Context` in the `Parser` and the reference to the string
slice in the `Context` have different lifetimes, and we’ve ensured that the
lifetime of the string slice is longer than the reference to the `Context`.

That was a very long-winded example, but as we mentioned at the start of this
chapter, these features are pretty niche. You won’t often need this syntax, but
it can come up in situations like this one, where you need to refer to
something you have a reference to.

### Lifetime Bounds on References to Generic Types

In the “Trait Bounds” section of Chapter 10, we discussed using trait bounds on
generic types. We can also add lifetime parameters as constraints on generic
types, and these are called *lifetime bounds*. Lifetime bounds help Rust verify
that references in generic types won’t outlive the data they’re referencing.

<!-- Can you say up front why/when we use these? -->
<!-- Done -->

For an example, consider a type that is a wrapper over references. Recall the
`RefCell<T>` type from the “`RefCell<T>` and the Interior Mutability Pattern”
section of Chapter 15: its `borrow` and `borrow_mut` methods return the types
`Ref` and `RefMut`, respectively. These types are wrappers over references that
keep track of the borrowing rules at runtime. The definition of the `Ref`
struct is shown in Listing 19-16, without lifetime bounds for now:

<span class="filename">Filename: src/lib.rs</span>

```rust,ignore
struct Ref<'a, T>(&'a T);
```

<span class="caption">Listing 19-16: Defining a struct to wrap a reference to a
generic type; without lifetime bounds to start</span>

Without explicitly constraining the lifetime `'a` in relation to the generic
parameter `T`, Rust will error because it doesn’t know how long the generic
type `T` will live:

```text
error[E0309]: the parameter type `T` may not live long enough
 --> src/lib.rs:1:19
  |
1 | struct Ref<'a, T>(&'a T);
  |                   ^^^^^^
  |
  = help: consider adding an explicit lifetime bound `T: 'a`...
note: ...so that the reference type `&'a T` does not outlive the data it points at
 --> src/lib.rs:1:19
  |
1 | struct Ref<'a, T>(&'a T);
  |                   ^^^^^^
```

Because `T` can be any type, `T` could itself be a reference or a type that
holds one or more references, each of which could have their own lifetimes.
Rust can’t be sure `T` will live as long as `'a`.

Fortunately, that error gave us helpful advice on how to specify the lifetime
bound in this case:

```text
consider adding an explicit lifetime bound `T: 'a` so that the reference type
`&'a T` does not outlive the data it points at
```

Listing 19-17 shows how to apply this advice by specifying the lifetime bound
when we declare the generic type `T`.

```rust
struct Ref<'a, T: 'a>(&'a T);
```

<span class="caption">Listing 19-17: Adding lifetime bounds on `T` to specify
that any references in `T` live at least as long as `'a`</span>

This code now compiles because the `T: 'a` syntax specifies that `T` can be any
type, but if it contains any references, the references must live at least as
long as `'a`.

We could solve this in a different way, shown in the definition of a
`StaticRef` struct in Listing 19-18, by adding the `'static` lifetime bound on
`T`. This means if `T` contains any references, they must have the `'static`
lifetime:

```rust
struct StaticRef<T: 'static>(&'static T);
```

<span class="caption">Listing 19-18: Adding a `'static` lifetime bound to `T`
to constrain `T` to types that have only `'static` references or no
references</span>

Because `'static` means the reference must live as long as the entire program,
a type that contains no references meets the criteria of all references living
as long as the entire program (because there are no references). For the borrow
checker concerned about references living long enough, there’s no real
distinction between a type that has no references and a type that has
references that live forever; both of them are the same for the purpose of
determining whether or not a reference has a shorter lifetime than what it
refers to.

### Inference of Trait Object Lifetimes

In Chapter 17 in the “Using Trait Objects that Allow for Values of Different
Types” section, we discussed trait objects, consisting of a trait behind a
reference, that allow us to use dynamic dispatch. We haven’t yet discussed what
happens if the type implementing the trait in the trait object has a lifetime
of its own. Consider Listing 19-19, where we have a trait `Red` and a struct
`Ball`. `Ball` holds a reference (and thus has a lifetime parameter) and also
implements trait `Red`. We want to use an instance of `Ball` as the trait
object `Box<Red>`:

<span class="filename">Filename: src/main.rs</span>

```rust
trait Red { }

struct Ball<'a> {
    diameter: &'a i32,
}

impl<'a> Red for Ball<'a> { }

fn main() {
    let num = 5;

    let obj = Box::new(Ball { diameter: &num }) as Box<Red>;
}
```

<span class="caption">Listing 19-19: Using a type that has a lifetime parameter
with a trait object</span>

This code compiles without any errors, even though we haven’t said anything
explicit about the lifetimes involved in `obj`. This works because there are
rules having to do with lifetimes and trait objects:

* The default lifetime of a trait object is `'static`.
* With `&'a Trait` or `&'a mut Trait`, the default lifetime is `'a`.
* With a single `T: 'a` clause, the default lifetime is `'a`.
* With multiple `T: 'a`-like clauses, there is no default; we must
  be explicit.

When we must be explicit, we can add a lifetime bound on a trait object like
`Box<Red>` with the syntax `Box<Red + 'a>` or `Box<Red + 'static>`, depending
on what’s needed. Just as with the other bounds, this means that any
implementor of the `Red` trait that has references inside must have the
same lifetime specified in the trait object bounds as those references.

Next, let’s take a look at some other advanced features dealing with traits!
