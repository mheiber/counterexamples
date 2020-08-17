# Incomplete variance checking

With declaration-site variance (see [Covariant
containers](covariant-containers.md)), it is possible to define a
class `C` with a covariant parameter `A`, so that `C[X]` is a subtype
of `C[Y]` whenever `X` is a subtype of `Y`. However, when typechecking
`C` it is important to verify that all uses of the parameter `A` are
actually compatible with covariance. Many soundness issues have arisen
from skipping such checks:

  - **Private members** may not need checking for variance, but this
      is subtle. See [Privacy violations](privacy-violations.md).

  - **Constructor parameters** do not normally need to be checked for
    covariance, as they do not form part of an object's
    interface. However, this does not apply to parameters of the
    constructor of an inner class that can access members of its
    enclosing class. For instance, in Scala[^scala-param]:

    ```scala
    // Counterexample by Martin Odersky
    class C[+A] {
      private[this] var y: A = _
      def getY: A = y
      class Inner(x: A) {
        y = x
      }
    }
     
    object Test {
      def main(args: Array[String]) = {
        val x = new C[String]
        val y: C[Any] = x
        val i = new y.Inner(1)
        val s: String = x.getY
        println(s)
      }
    }
    ```

  - **GADT parameters** introduce equations between types when matched
    on, making it unsound to mark them as either covariant or
    contravariant. This issue showed up in early versions of OCaml and
    in Scala[^scala-gadt]:
    ```ocaml
    (* Counterexample by Jeremy Yallop *)
    let magic : 'a 'b. 'a -> 'b =
      fun (type a) (type b) (x : a) ->
        let bad_proof (type a) =
          (Refl : (< m : a>, <m : a>) eq :> (<m : a>, < >) eq) in
        let downcast : type a. (a, < >) eq -> < > -> a =
          fun (type a) (Refl : (a, < >) eq) (s : <>) -> (s :> a) in
        (downcast bad_proof ((object method m = x end) :> < >)) # m
    ```
    ```scala
    // Counterexample by Owen Healy
    object Test extends App {
      sealed trait Node[+A]
      case class L[C,D](f: C => D) extends Node[C => D]
    
      def test[A,B](n: Node[A => B]): A => B = n match {
        case l: L[c,d] => l.f
      }
    
      println {
        test(new L[Int,Int](identity) with
          Node[Nothing]: Node[Int => String])(3): String
      }
    }
    ```
    The details of combining GADTs and declaration-site variance are
    tricky. Giarusso[^gadt-giarusso] describes the problem in detail,
    and Scherer and Rémy[^gadt-scherer] identify several cases where
    they can be soundly combined.

  - **Self types** allow a class to refer to the type of `this` (which
    may be a subtype of the class being defined). However, if the
    class has type parameters, any use of a self type must count as a
    use of those parameters. Failure to do so led to a soundness issue
    in Hack[^hack-self]:
    ```hack
    // Counterexample by Derek Lam
    class Base {}
    class Derived extends Base {
        public function __construct(public int $derived_prop) {}
    }
    final class ImplCov<+T> {
        public function __construct(private T $v) {}
        public function put(this $v): void {
            $this->v = $v->v;
        }
        public function pull(): T {
            return $this->v;
        }
    }
    class Violate {
        public static function foo(ImplCov<Derived> $v): void {
            self::bar($v);
            echo $v->pull()->derived_prop;
            // Wait... Base doesn't have $derived_prop!
        }
        public static function bar(ImplCov<Base> $v): void {
            $v->put(new ImplCov(new Base()));
        }
    }
    // Violate::foo(new ImplCov(new Derived(42))); crashes
    ```

  - **Subclasses** may introduce new methods which do not respect
    covariance, producing an invariant subclass of a covariant
    class. This is not by itself unsound, but can cause unsoundness if
    the language also supports downcasting by pattern-matching,
    allowing covariant use of the invariant derived class through the
    covariant base class. This problem arose in Scala[^scala-case]:
    ```scala
    // Counterexample by Chung-Kil Hur
    sealed abstract class MyADT[+A]
    case object MyNone extends MyADT[Nothing]
    case class MyFun[A](fun: A=>A) extends MyADT[A]
    
    def data : MyADT[Any] = MyFun((x:Int)=>x+1)
    
    val foo : Any =
      data match {
        case MyFun(f) => f("a")
        case _ => 0
      }
    ```

[^scala-param]: <https://github.com/scala/bug/issues/9549> (2015)

[^scala-gadt]: <https://github.com/scala/bug/issues/8563> (2014)

[^gadt-giarusso]:
[Open GADTs and Declaration-site Variance: A Problem Statement](http://lampwww.epfl.ch/~hmiller/scala2013/resources/pdfs/paper5.pdf),
Paolo G. Giarrusso (2013)

[^gadt-scherer]:
[GADTs meet subtyping](https://arxiv.org/abs/1301.2903) (ESOP '13)
Gabriel Scherer and Didier Rémy (2013)

[^hack-self]: <https://github.com/facebook/hhvm/issues/7254> (2016)

[^scala-case]: <https://github.com/scala/bug/issues/8737#issuecomment-292432742> (2016)