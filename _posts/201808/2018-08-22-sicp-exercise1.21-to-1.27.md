# sicp 1.21

## exercise 1.21
<pre><code>(define (smallest-divisor n)
	(fine-divisor n 2))
	
(define (fine-divisor n test-divisor)
	(cond ((> (square test-divisor) n) n)
			((divides? test-divisor n) test-divisor)
			(else (fine-divisor n (+ test-divisor 1)))))
			
(define (divides? a b)
	(= (remainder b a ) 0))
	
(define (square x)
  (* x x))
  
(define (prime? n)
	(= (smallest-divisor n) n))</code></pre>
  
## exercise 1.22

<pre><code>(load "smallest-divisor.scm")

(define (search-for-primes n min-count)
(define (timed-prime-test n)
	(newline)
	(display n)
	(start-prime-test n (runtime)))	
(define (start-prime-test n start-time)
	(if (prime? n)
		(report-prime (- (runtime) start-time))))		
(define (report-prime elapsed-time)
	(display " *** ")
	(set! min-count (- min-count 1))
	(display elapsed-time))
	(timed-prime-test (+ n 1))
	(if (< 0 min-count) 
	(search-for-primes (+ n 2) min-count)
	)
)</code></pre>

符合预期，结合1.23 1.24一起看。
当输入n越大的时候，计算素数消耗的时候越接近理论值。n比较小的时候，消耗时候方差较大。

## exercise 1.23
速度有提升，但也只是提升了一点点，远远没有2倍差距。

## exercise 1.24
计算速度差距不大，说明影响计算速度的因素很多，步数只是因素之一。

## exercise 1.25
不一样，习题中的方法将会产生出很大的数。

## exercise 1.26
将square展开之后，每一次expmod都会迭代产生两次expmod，所以时间复杂度又回去了。

## exercise 1.27
这些数成功的骗过了fast-prime 但是无法骗过prime

## exercise 1.28
以后有空了再做吧。