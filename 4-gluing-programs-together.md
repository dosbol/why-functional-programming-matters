# Gluing programs together

The other new kind of glue that functional languages provide enables whole
programs to be glued together. Recall that a complete functional program is
just a function from its input to its output. If f and g are such programs, then
(g . f ) is a program that, when applied to its input, computes

g (f input)

The program f computes its output, which is used as the input to program g.
This might be implemented conventionally by storing the output from f in a
temporary file. The problem with this is that the temporary file might occupy
so much memory that it is impractical to glue the programs together in this way.
Functional languages provide a solution to this problem. The two programs f
and g are run together in strict synchronization. Program f is started only
when g tries to read some input, and runs only for long enough to deliver the
output g is trying to read. Then f is suspended and g is run until it tries to read
another input. As an added bonus, if g terminates without reading all of f ’s
output, then f is aborted. Program f can even be a nonterminating program,
producing an infinite amount of output, since it will be terminated forcibly as
soon as g is finished. This allows termination conditions to be separated from
loop bodies — a powerful modularization.
Since this method of evaluation runs f as little as possible, it is called “lazy
evaluation”. It makes it practical to modularize a program as a generator that
constructs a large number of possible answers, and a selector that chooses the
appropriate one. While some other systems allow programs to be run together
in this manner, only functional languages (and not even all of them) use lazy
evaluation uniformly for every function call, allowing any part of a program to
be modularized in this way. Lazy evaluation is perhaps the most powerful tool
for modularization in the functional programmer’s repertoire.
We have described lazy evaluation in the context of functional languages,
but surely so useful a feature should be added to nonfunctional languages —
or should it? Can lazy evaluation and side-effects coexist? Unfortunately, they
cannot: Adding lazy evaluation to an imperative notation is not actually impossible, but the combination would make the programmer’s life harder, rather than
easier. Because lazy evaluation’s power depends on the programmer giving up
any direct control over the order in which the parts of a program are executed,
it would make programming with side effects rather difficult, because predicting
in what order —or even whether— they might take place would require knowing
a lot about the context in which they are embedded. Such global interdependence would defeat the very modularity that —in functional languages— lazy
evaluation is designed to enhance.

## 4.1  Newton-Raphson Square Roots

We will illustrate the power of lazy evaluation by programming some numerical
algorithms. First of all, consider the Newton-Raphson algorithm for finding
square roots. This algorithm computes the square root of a number n by starting
from an initial approximation a0 and computing better and better ones using
the rule

ai+1 = (ai + n/ai )/2

If the approximations converge to some limit a, then

a = (a + n/a)/2

so

2a = a + n/a
a = n/a
a∗a = n
a = √n

In fact the approximations converge rapidly to a limit. Square root programs
take a tolerance (eps) and stop when two successive approximations differ by
less than eps.
The algorithm is usually programmed more or less as follows:

C N IS CALLED ZN HERE SO THAT IT HAS THE RIGHT TYPE
  X = A0
  Y = A0 + 2. * EPS
C Y’S VALUE DOES NOT MATTER SO LONG AS ABS(X-Y).GT.EPS
100 IF ABS(X-Y).LE.EPS GOTO 200
  Y = X
  X = (X + ZN/X) / 2.
GOTO 100
200 CONTINUE
C THE SQUARE ROOT OF ZN IS NOW IN X.

This program is indivisible in conventional languages. We will express it in a
more modular form using lazy evaluation and then show some other uses to
which the parts may be put.
Since the Newton-Raphson algorithm computes a sequence of approximations it is natural to represent this explicitly in the program by a list of approximations. Each approximation is derived from the previous one by the function

next n x = (x + n/x)/2

so (next n) is the function mapping one approximation onto the next. Calling
this function f , the sequence of approximations is

[a0, f a0, f (f a0), f (f (f a0)), . . . ]

We can define a function to compute this:

repeat f a = Cons a (repeat f (f a))

so that the list of approximations can be computed by

repeat (next n) a0

The function repeat is an example of a function with an “infinite” output — but
it doesn’t matter, because no more approximations will actually be computed
than the rest of the program requires. The infinity is only potential: All it means
is that any number of approximations can be computed if required; repeat itself
places no limit.
The remainder of a square root finder is a function within, which takes a
tolerance and a list of approximations and looks down the list for two successive
approximations that differ by no more than the given tolerance. It can be
defined by

within eps (Cons a (Cons b rest))
  = b, if abs (a − b) ≤ eps
  = within eps (Cons b rest), otherwise

Putting the parts together, we have

sqrt a0 eps n = within eps (repeat (next n) a0)

Now that we have the parts of a square root finder, we can try combining
them in different ways. One modification we might wish to make is to wait
for the ratio between successive approximations to approach 1, rather than for
the difference to approach 0. This is more appropriate for very small numbers
(when the difference between successive approximations is small to start with)
and for very large ones (when rounding error could be much larger than the
tolerance). It is only necessary to define a replacement for within:

relative eps (Cons a (Cons b rest))
  = b, if abs (a/b − 1) ≤ eps
  = relative eps (Cons b rest), otherwise

Now a new version of sqrt can be defined by

relativesqrt a0 eps n = relative eps (repeat (next n) a0)

It is not necessary to rewrite the part that generates approximations.

## 4.2 Numerical Differentiation

We have reused the sequence of approximations to a square root. Of course,
it is also possible to reuse within and relative with any numerical algorithm
that generates a sequence of approximations. We will do so in a numerical
differentiation algorithm.
The result of differentiating a function at a point is the slope of the function’s
graph at that point. It can be estimated quite easily by evaluating the function
at the given point and at another point nearby and computing the slope of a
straight line between the two points. This assumes that if the two points are
close enough together, then the graph of the function will not curve much in
between. This gives the definition

easydiff f x h = (f (x + h) − f x)/h

In order to get a good approximation the value of h should be very small.
Unfortunately, if h is too small then the two values f (x + h) and f (x) are very
close together, and so the rounding error in the subtraction may swamp the
result. How can the right value of h be chosen? One solution to this dilemma
is to compute a sequence of approximations with smaller and smaller values of
h, starting with a reasonably large one. Such a sequence should converge to the
value of the derivative, but will become hopelessly inaccurate eventually due to
rounding error. If (within eps) is used to select the first approximation that
is accurate enough, then the risk of rounding error affecting the result can be
much reduced. We need a function to compute the sequence:

differentiate h0 f x = map (easydiff f x) (repeat halve h0)
halve x = x/2

Here h0 is the initial value of h, and successive values are obtained by repeated
halving. Given this function, the derivative at any point can be computed by

within eps (differentiate h0 f x)

Even this solution is not very satisfactory because the sequence of approximations converges fairly slowly. A little simple mathematics can help here. The
elements of the sequence can be expressed as

the right answer + an error term involving h

and it can be shown theoretically that the error term is roughly proportional to
a power of h, so that it gets smaller as h gets smaller. Let the right answer be A,
and let the error term be B × hn . Since each approximation is computed using
a value of h twice that used for the next one, any two successive approximations
can be expressed as

ai = A + B × 2n × hn
and
ai+1 = A + B × hn

Now the error term can be eliminated. We conclude

A= (an+1 × 2n − an) / 2n − 1

Of course, since the error term is only roughly a power of h this conclusion is
also approximate, but it is a much better approximation. This improvement
can be applied to all successive pairs of approximations using the function

elimerror n (Cons a (Cons b rest))
= Cons ((b ∗ (2^n) − a)/(2^n − 1)) (elimerror n (Cons b rest))

Eliminating error terms from a sequence of approximations yields another sequence, which converges much more rapidly.
One problem remains before we can use elimerror — we have to know the
right value of n. This is difficult to predict in general but is easy to measure.
It’s not difficult to show that the following function estimates it correctly, but
we won’t include the proof here:

order (Cons a (Cons b (Cons c rest)))
= round (log2 ((a − c)/(b − c) − 1))
round x = x rounded to the nearest integer
log2 x = the logarithm of x to the base 2

Now a general function to improve a sequence of approximations can be defined:

improve s = elimerror (order s) s

The derivative of a function f can be computed more efficiently using improve,
as follows:

within eps (improve (differentiate h0 f x))

The function improve works only on sequences of approximations that are computed using a parameter h, which is halved for each successive approximation.
However, if it is applied to such a sequence its result is also such a sequence!
This means that a sequence of approximations can be improved more than once.
A different error term is eliminated each time, and the resulting sequences converge faster and faster. Hence one could compute a derivative very efficiently
using

within eps (improve (improve (improve (differentiate h0 f x))))

In numerical analysts’ terms, this is likely to be a fourth-order method, and it
gives an accurate result very quickly. One could even define

super s = map second (repeat improve s)
second (Cons a (Cons b rest)) = b

which uses repeat improve to get a sequence of more and more improved sequences of approximations and constructs a new sequence of approximations
by taking the second approximation from each of the improved sequences (it
turns out that the second one is the best one to take — it is more accurate than the first and doesn’t require any extra work to compute). This algorithm
is really very sophisticated — it uses a better and better numerical method as
more and more approximations are computed. One could compute derivatives
very efficiently indeed with the program:

within eps (super (differentiate h0 f x))

This is probably a case of using a sledgehammer to crack a nut, but the point
is that even an algorithm as sophisticated as super is easily expressed when
modularized using lazy evaluation.

## 4.3 Numerical Integration

The last example we will discuss in this section is numerical integration. The
problem may be stated very simply: Given a real-valued function f of one real
argument, and two points a and b, estimate the area under the curve that f
describes between the points. The easiest way to estimate the area is to assume
that f is nearly a straight line, in which case the area would be

easyintegrate f a b = (f a + f b) ∗ (b − a)/2

Unfortunately this estimate is likely to be very inaccurate unless a and b are
close together. A better estimate can be made by dividing the interval from a to
b in two, estimating the area on each half, and adding the results. We can define
a sequence of better and better approximations to the value of the integral by
using the formula above for the first approximation, and then adding together
better and better approximations to the integrals on each half to calculate the
others. This sequence is computed by the function

integrate f a b = Cons (easyintegrate f a b)
(map addpair (zip2 (integrate f a mid )
(integrate f mid b)))
where mid = (a + b)/2

The function zip2 is another standard list-processing function. It takes two lists
and returns a list of pairs, each pair consisting of corresponding elements of the
two lists. Thus the first pair consists of the first element of the first list and the
first element of the second, and so on. We can define zip2 by

zip2 (Cons a s) (Cons b t) = Cons (a, b) (zip2 s t)

In integrate, zip2 computes a list of pairs of corresponding approximations to
the integrals on the two subintervals, and map addpair adds the elements of the
pairs together to give a list of approximations to the original integral.
Actually, this version of integrate is rather inefficient because it continually
recomputes values of f . As written, easyintegrate evaluates f at a and at b,
and then the recursive calls of integrate re-evaluate each of these. Also, (f mid)
is evaluated in each recursive call. It is therefore preferable to use the following
version, which never recomputes a value of f :

integrate f a b = integ f a b (f a) (f b)
integ f a b f a f b =

Cons ((f a + f b) ∗ (b − a)/2)
map addpair(zip2 (integ f a m f a f m)
(integ f m b f m f b)))
where m = (a + b)/2
fm = f m

The function integrate computes an infinite list of better and better approximations to the integral, just as differentiate did in the section above. One can
therefore just write down integration routines that integrate to any required
accuracy, as in

within eps (integrate f a b)
relative eps (integrate f a b)

This integration algorithm suffers from the same disadvantage as the first differentiation algorithm in the preceding subsection — it converges rather slowly.
Once again, it can be improved. The first approximation in the sequence is
computed (by easyintegrate) using only two points, with a separation of b − a.
The second approximation also uses the midpoint, so that the separation between neighboring points is only (b − a)/2. The third approximation uses this
method on each half-interval, so the separation between neighboring points is
only (b − a)/4. Clearly the separation between neighboring points is halved
between each approximation and the next. Taking this separation as h, the
sequence is a candidate for improvement using the function improve defined
in the preceding section. Therefore we can now write down quickly converging
sequences of approximations to integrals, for example,

super (integrate sin 0 4)
and
improve (integrate f 0 1)
where f x = 1/(1 + x ∗ x)

(This latter sequence is an eighth-order method for computing π/4. The second
approximation, which requires only five evaluations of f to compute, is correct
to five decimal places.)
In this section we have taken a number of numerical algorithms and programmed them functionally, using lazy evaluation as glue to stick their parts
together. Thanks to this, we have been able to modularize them in new ways,
into generally useful functions such as within, relative, and improve. By combining these parts in various ways we have programmed some quite good numerical algorithms very simply and easily.

