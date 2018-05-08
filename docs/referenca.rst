Alder Manual
============

.. 2018 0507

This document describes the Alder programming language.

Alder is an imperative programming language.  Its syntax and semantics
show influences from ALGOL-60, Pascal, and other languages.  I hope to
use Alder as a test-bed for various ideas I've had over the years.

High level goals
----------------

* in general, make scoping work so that a procedure can be understood
  in isolation (best case) or with minimum reference to other entities
  (worst case)
* provide a simple module mechanism for separate compilation
* detect various nonsensical runtime conditions

Ideas::

 1. Like Gypsy, disallow global mutable state.
 2. For a variable, the scope is exactly one of:
    1. A local variable in the current procedure.
    2. A parameter in the current procedure.
 1. For a constant (I.e. a named constant, a named type, or a procedure), the scope is exactly one of:
    1. Local to the current procedure
    2. Local to the current module
    3. Imported from a foreign module.  Note that imports are not transitive.
 1. Like Pascal, provide some relatively machine-independent basic types, and a way to construct composite types (records).  Unlike Pascal, Alder does not provide pointers.
 2. Like CSP, provide a way to do concurrent programming without shared state.
 3. Like Go and modern Fortran, provide ways to resize arrays at runtime.
    1. example declaration: integer A[1:9]
    2. example array extension: append A, 1;  (array A now has 10 elements.) 
    3. example array downward resize: A := A[1:4] (now A has 4 elements.)  you can't resize an array to make it bigger; only append can do that.
    4. Like a Go slice, an array has a size and a capacity.  The size goes down when you do a resize, but the capacity is unchanged.  capacity grows via append, as in Go.
 1.  provide ways to recognize various nonsensical situations:
    1. dead read - Read before write, i.e. uninitialized read of a variable.
    2. dead write - Write after write (double write, first write is useless.  This is similar to Go's "must read" detector.)
    3. terminal write - write to loc var not followed by a read, useless.
    4. exit no write – exit function without writing to output parameter
    5. write to constant - write to an in-only parameter.
    6. read from output – read from an out-only parameter.
    7. dynamic memory allocation problems (read before alloc, read after alloc but before init, read after free, double free)
    8. out-of-bounds array references.
 1. how parameters work:  
    1. For regular procedures, parameters are passed in-out.  One way this could be implemented is by using pass-by-reference.
    2. For function procedures, input parameters are passed in-only, and output parameters are passed out-only.  In both cases, the parameters can use pass-by-reference, because input parameters cannot appear on the LHS of an assignment, and output parameters cannot appear on the RHS of an assignment or in an expression context. 
    3. Function procedures return their results via explicit output parameters.  A function procedure may return multiple results.
    4. For both regular and function procedures, parameters may not be aliased.  An implementation is required to detect such aliasing and flag it as an error. This detection may be at compile time or at runtime. 
 1. Procedure declarations may not be nested.  The one exception is that a procedure that takes a procedure parameter, includes a "stub declaration" for the procedure parameter.  This stub declaration provides just the types of the procedure parameter's own input and output parameters, and otherwise contains no executable statements.
 2. Top level compilation units include a nameless block (the program's initial procedure) and zero or more modules.  
 3. A module is a special kind of block with no statements and with no variable declarations.  A module only includes constant, type, and procedure declarations.  All the entities declared in the module are exported.  A client module or block may access these entities by prefixing an exported entity with its module name.
 4. Similar to how this works in Go, string is just sugar for byte array, with the constraint that the contents will be interpretable as ASCII (sans char 127).  Like all arrays, a string knows its own length, available via the size() intrinsic.


High level structures --------------------------------------------------------------


compilation-unit = module | block .
module = module name “;” block .
block = begin declaration-list (";" statement-list )?  end .


Declarations --------------------------------------------------------------------------


declaration = constant-declaration | type-declaration | variable-declaration | procedure-declaration | use-declaration .
declaration-list = declaration (";" declaration)* .
constant-declaration = constant type name “=” expression .
type-declaration = type name = type .
variable-declaration = type name-list .
type = simple-type | compound-type | type-name .
simple-type = integer | real | boolean | byte .
compound-type = ( simple-type | record-type ) array-type | record-type | string .
array-type = array bounds .
bounds = “[“ (bound-pair (“,” bound-pair)*)? “]”.
bound-pair = ( expression “:” expression ) | "*" .
record-type = record “(“ variable-declaration (“,” variable-declaration)* “)”.
procedure-declaration = procedure name “(“ name-list “)” returns? declaration-list block .
returns = returns “(“ name-list “)” ) .
use-declaration = use module-name (“,” module-name)*. 


Statements ------------------------------------------------------------------------------


statement-list = statement (";" statement)*.
statement = assignment | if | for | while | break | continue | compound-statement | call .
assignment = target “:=” (expression | display).
target = name ( atail | rtail )? .
atail = "[" expr-list "]" rtail? . 
rtail = “.” target.
display = ( array “(“ (expression-list | all expression) “)” ) | ( record “(“ expression-list “)” ).
for = for variable “:=” expression step expression until expression do statement .
while = while expression do statement .
if = if expression then statement ( else statement )? . 
break = break .
continue = continue . 
compound-statement = begin statement (“;” statement)* end .
call = procedure-name “(“ expression-list “)” .


Expressions -------------------------------------------------------------------------------


expression = simple-expression (relation simple-expression)?.
relation = “=” | “<>” | “<” | “>” | “<=” | “>=” .
simple-expression = (“+”|”-”)? term (adding-operator term)*.
adding-operator = “+” | “-” | or .
term = factor (multiplying-operator factor)*. 
multiplying-operator = “*” | “/” | div | rem | and .
factor = number | string | true | false | “(“ expression “)” | not factor | call .


<end of syntax>


Examples.


-- example module exporting M.pi, M.double, M.swap 
module M;
begin
        constant real pi = 3.14;
        procedure double(x) returns (y); 
        begin
                real x, y;
                y := 2 * x
        end;
        procedure rswap(x, y);
        begin 
                real x, y, t;
                t := x; 
                x := y; 
                y := t
        end
end
-- example of a function procedure with a procedure argument 
procedure integrate(f, lo, hi, n) returns (a);
begin
        procedure f(x) returns (y);
                begin real x,y end;
        -- note stub declaration.  there are no nested procedure declarations.
        real lo, hi, a, width, y, area;
        area := 0.0;
        width := (hi - lo)/n;
        x = lo + (width/2);
        while x < hi do 
        begin
                y := f(x);
                rect := y * width; 
                area := area + rect
        end
        a := area
end


Examples of using displays.


integer array D[1:5], E[1:100];
D := array(3,1,4,1,5);
E := array( all 0 ); 
record(boolean b, real r) S;
S := record(true, 1.0); 


Idea – Pascal-PLUS's inner statement.


type url_record = record ( string scheme );
type host_record = url_record + record( string host ); 
procedure print_URL( url_record u ) ;
begin  
print(u.scheme);
inner
end;


procedure print_URL( host_record h );
begin
print(h.host)
end;


url_record R;
R := record("http", "foo.com");
print_URL( R ) ==> prints http://foo.com


How does one wrap that BETA-like idea in an ALGOL-like syntax?
(2018 0501 – defer the type extension for now.)



[end of file]
