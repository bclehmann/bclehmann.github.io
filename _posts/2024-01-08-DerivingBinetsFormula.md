---
layout: post
title:  "Binet's Formula for Fibonacci Numbers and Where it Comes From"
---

## Prologue

If you're like me, you may be a little jaded towards the Golden Ratio and the Fibonacci numbers. If you aren't already, I invite you to search YouTube for videos related to either topic &mdash; I had a look and I was suggested a video entitled 
"The Golden Ratio - Transmuted Pain to Power in Infinite Divine Proportion". Suffice it to say, there's a lot of garbage out there. 

The Golden Ratio was familiar to the Greeks, and much of its allure comes from Greek philosopher's obsession with the number. An allure that is evermore powerful today, as all three Abrahamic religions were greatly influenced by Greek philosophy, 
especially by the Neoplatonists. Unfortunately, while I'm sure it is possible to have an enlightening discussion about the role of the Golden Ratio in Greek philosophy and religion, much of the content concerning Phi makes a mockery of 
both philosophy and theology. To say nothing of mathematics.

That means that this article is not for you if you're here because you expected me to tell you how to cure cancer by using 1.618 cups of flour instead of 1.5. Instead, I'd like to take a poke at why the Golden Ratio and the Fibonacci Numbers are 
related in the first place, while also making a short review piece for those studying linear algebra.

## The Math

You may be familiar with the Binet Formula for the Fibonacci numbers:

$$ F_n = \frac{\varphi^n - (-\varphi)^{-n}}{\sqrt 5} $$

If not, you should watch [this video](https://www.youtube.com/watch?v=ghxQA3vvhsk), it's really quite a nice exploration of the formula.

But where does this formula come from? Well, it comes from observing that we can recontextualize the familiar Fibonacci series in matrix form:

$$ F_{n+1} = F_{n} + F_{n-1} = \begin{bmatrix} 1 & 1 \end{bmatrix} \begin{bmatrix} F_{n} \\ F_{n-1} \end{bmatrix} $$

This version is quite simple, but it's also quite uninteresting. However, if one adds another row to our matrix we will have a square matrix $ \mathbf A $, which unlocks a great many tools of linear algebra. Our second row will simply compute the previous Fibonacci number:

$$ \begin{bmatrix} F_{n+1} \\ F_n \end{bmatrix} = \begin{bmatrix} 1 & 1 \\ 1 & 0 \end{bmatrix} \begin{bmatrix} F_{n} \\ F_{n-1} \end{bmatrix} = \begin{bmatrix} 1 & 1 \\ 1 & 0 \end{bmatrix}^n \begin{bmatrix} 1 \\ 0 \end{bmatrix} $$

This is nice and all, but we want a more efficient way to compute this, not just a more mathy way. Fortunately, we can have both; we'll start by finding the eigenvalues of this matrix, which are the roots of the characteristic polynomial:

$$ p_{\mathbf A}(\lambda) = \begin{vmatrix} 1 - \lambda & 1 \\ 1 & -\lambda \end{vmatrix} $$

$$ = (1 - \lambda) \cdot (-\lambda) - 1 \cdot 1 $$

$$ = \lambda^2 - \lambda - 1 $$

This has roots of $ \frac{1 \pm \sqrt 5}{2} $, which we typically write as $ \varphi $ and $ -\frac{1}{\varphi} $. The corresponding eigenvectors are non-zero solutions to $ (\mathbf A - \lambda \mathbf I)\mathbf{x} = 0 $ for an eigenvalue $ \lambda $. 

We find the eigenvector associated with $ \lambda = \varphi $ by row-reduction on the following matrix:

$$ \left[\begin{matrix}\mathbf A - \varphi \mathbf I \end{matrix}\right. \left|\begin{matrix} 0 \end{matrix}\right] $$

$$ = \left[\begin{matrix} 1 - \varphi & 1 \\ 1 & -\varphi \end{matrix}\right. \left| \begin{matrix} \:0\: \\ \:0\: \end{matrix} \right] $$

Which is row-equivalent to:

$$ \left[\begin{matrix} 1 & -\varphi \\ 0 & 0 \end{matrix}\right. \left| \begin{matrix} \:0\: \\ \:0\: \end{matrix} \right] $$

This matrix encodes a system with the following solution set:

$$ \overrightarrow x = s \begin{bmatrix} \varphi \\ 1 \end{bmatrix} $$

Therefore an eigenvector associated with $ \lambda = \varphi $ is $ \overrightarrow u = \langle \varphi, 1 \rangle $. Similarly, an eigenvector associated with the other eigenvalue is $ \overrightarrow v = \langle -\varphi^{-1}, 1 \rangle $. 
These choices are not unique, we could've chosen any of the infinite parallel vectors (except for the zero vector).

As we have two eigenvectors from different eigenspaces (i.e. are associated with different eigenvalues) we know they are linearly independent, and thus form a basis of $ \mathbb{R}^2 $. The vector $ \langle 1, 0 \rangle $ is thus necessarily a linear 
combination of this eigenbasis:

$$ \begin{bmatrix} 1 \\ 0 \end{bmatrix} = \frac{1}{\sqrt 5}\overrightarrow u - \frac{1}{\sqrt 5}\overrightarrow v $$

Therefore, if we multiply by $ \mathbf A $ we only need to multiply each coefficient by the respective eigenvalue:

$$ \mathbf A \begin{bmatrix} 1 \\ 0 \end{bmatrix} = \mathbf A (\frac{1}{\sqrt 5}\overrightarrow u - \frac{1}{\sqrt 5}\overrightarrow v) $$

$$ = \frac{1}{\sqrt 5} \mathbf A \overrightarrow u - \frac{1}{\sqrt 5} \mathbf A \overrightarrow v $$

$$ = \frac{1}{\sqrt 5} \cdot \varphi \cdot \overrightarrow u - \frac{1}{\sqrt 5} \cdot (-\varphi)^{-1} \cdot \overrightarrow v $$

$$ = \frac{\varphi \: \overrightarrow u - (-\varphi)^{-1} \:\: \overrightarrow v}{\sqrt 5} $$

And if we multiply by $ \mathbf A^n $:

$$ \mathbf A^n \begin{bmatrix} 1 \\ 0 \end{bmatrix} = \frac{\varphi^n \: \overrightarrow u - (-\varphi)^{-n} \:\: \overrightarrow v}{\sqrt 5} $$

This lets us create a closed-form expression by solving for $ F_n $ which if you recall is the *second* element in our vector:

$$ F_n = \frac{\varphi^n - (-\varphi)^{-n}}{\sqrt 5} $$

And so there we have it, we've derived Binet's formula.
