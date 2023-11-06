---
title: "daBits and Domain Conversion in MP-SPDZ"
date: 2023-10-31
draft: false
math: true
editPost:
  URL: "https://github.com/hdvanegasm/hdvanegasm.github.io/tree/master/content"
  Text: "Suggest Changes"
  appendFilePath: true
---

In secure multi-party computation (MPC), each protocol has an underlying domain of computation in which the secure operations are performed. Such domains of computation are algebraic structures such as fields or rings. Some common examples for those domains are finite fields of order $q$, with $q$ prime, and rings of the form $\mathbb{Z}_{2^k}$. A special case is the field $\mathbb{Z}_2$, which is the one that we refer to as *binary domain*. For the cases where the field has order $q > 2$ and $\mathbb{Z}\_{2^k}$ with $k > 1$, we refer to them as *arithmetic domains*.

Each domain has its advantages for some given tasks. For example, arithmetic domains are widely used in tasks where there are a significant amount of integer operations such as computing statistics over some dataset. On the other hand, binary domains are preferred when there is a significant amount of operations that go down to a bit level of computation, for example, in integer comparisons and bit-wise operations (Damgård et al., 2019). However, some applications need the advantages of both worlds to achieve a better performance. For example, one may think of a program that needs a huge amount of bit-level computations but also needs to perform arithmetic computations between secret integers. In this case, a solution that fits better for this situation is a *mixed-circuit computation*.

In this blog post, we will see how to make the transition from secret values shared in a binary domain to shares in an arithmetic domain using daBits, and how to implement this in the MP-SDPZ framework. The purpose of this blog post is not to explain the underlying protocols to generate daBits given that it is a very extensive topic (such content may be a blog post by itself). Instead, we will explain briefly what are daBits along a basic protocol to do the domain conversion, and finally how to do this using the mentioned framework above.

This blog assumes that the reader is familiar with the following concepts: basic concepts on MPC, secret-sharing schemes, and MP-SDPZ basics (see the [tutorial](https://github.com/data61/MP-SPDZ/blob/master/Programs/Source/tutorial.mpc) and the [documentation](https://mp-spdz.readthedocs.io/en/latest/) of the framework). Also, it would be desirable previous exposure to daBits, although it is not completely needed given that I will present a very high-level definition.

### Notation

This blog post assumes that we are using general-purpose protocols based on linear secret-sharing schemes (SS) that realize an arithmetic black box functionality. This means that the protocols that we are considering allow the secure computation of additions and multiplications. We denote $\llbracket x \rrbracket \stackrel{\text{def}}{=} (x_1, x_2, \dots x_n)$ as the shares of a secret value $x$. Given that we are dealing with mixed-circuit computation, we denote $\llbracket x \rrbracket_2$ as a binary sharing of $x \in \mathbb{Z}_2$, meaning that all the shares $x_1, \dots, x_n$ are in $\mathbb{Z}_2$. Without loss of generality, we will consider that the arithmetic domain that we will use is $\mathbb{Z}_q$, for $q > 2$ a prime number. In this case, we write $\llbracket x \rrbracket_q$ meaning that the shares $x_1, \dots, x_n$ are in $\mathbb{Z}_q$.

With this notation, the goal of this blog post is to explain how to convert $\llbracket x \rrbracket_2$ into a sharing $\llbracket x \rrbracket_q$ for a given value $x \in \mathbb{Z}_2$.

## daBits and domain conversion

According to Rotaru & Wood (2019) and Aly et al. (2019), a *daBit* (which stands for **d**ouble **a**uthenticated **bit**) is a tuple of the form $(\llbracket r \rrbracket_2, \llbracket r \rrbracket_q)$, where $r \in \mathbb{Z}_2$ is a random bit that is unknown to all the parties participating in the protocol. Notice that, although the underlying shared value $r$ is the same in both the binary sharing and the arithmetic sharing, the shares in each case are not necessarily equal. Consider for example a protocol based on an additive secret-sharing scheme for 3 parties, and $q = 23$. If $r = 1$, it may happen that $\llbracket r \rrbracket_2 = (0, 0, 1)$ and $\llbracket r \rrbracket_q = (18, 5, 1)$. In both cases, the sum of all the shares in their corresponding algebraic structure is $r = 1$, but the underlying shares are completely different.

Given that the underlying value in a daBit is chosen at random and does not depend on the inputs provided by the parties, those daBits can be generated in a preprocessing stage of the protocol, that is, the daBits can be generated before each party provides its input to the protocol execution.

For a complete specification of a protocol that generates daBits, I refer the reader to the works of [Rotaru & Wood (2019)](https://eprint.iacr.org/2019/207) and [Aly et al. (2019)](https://eprint.iacr.org/2019/974).

Assuming that we can securely generate daBits, we can use them to design a protocol to convert binary shares into arithmetic shares. A protocol to accomplish this task, taken from [Damgård (2019)](https://eprint.iacr.org/2019/599), works as follows:

**Input:** a sharing $\llbracket x \rrbracket_2$ for $x \in \mathbb{Z}_2$.

**Output:** a sharing $\llbracket x \rrbracket_q$

1. Get a fresh daBit $(\llbracket r \rrbracket_2, \llbracket r \rrbracket_q)$ generated in the preprocessing phase.
2. Each party computes locally $\llbracket c \rrbracket_2 = \llbracket x \rrbracket_2 + \llbracket r \rrbracket_2$.
3. The parties reveal and learn the value of $c$.
4. The parties compute locally $\llbracket x \rrbracket_q = c - \llbracket r \rrbracket_q$.
5. The protocol outputs $\llbracket x \rrbracket_q$.

Looking at the protocol specification, in Step 3, the parties reveal $c$ but its value does not reveal anything about the value of $x$ because $r$ acts as a one-time pad. Also, the round complexity of this protocol is 1 round.

## How to do it in MP-SDPZ

Now that I have introduced some concepts, I will present how to transform shares from a binary domain to an arithmetic domain using daBits under the MP-SDPZ framework. The MP-SDPZ framework is a software to benchmark multiple MPC protocols under the same ground [(Keller, 2020)](https://eprint.iacr.org/2020/521). This framework is particularly useful to determine which techniques are best suited for a given task that needs to be computed securely. At a high level, the developer writes the functionality in language derived from Python 3; then, the software compiles the source code into a *bytecode* that is run by a virtual machine.

Given that the source code is passed through a compiler, in some situations, the compiler automatically detects when it is appropriate to use daBits. Consider the following example:

```python
program.use_edabit(True)

a = sint.get_input_from(0)
b = sint.get_input_from(1)

c = (a < b)

print_ln("%s", c.reveal())
```

This program is the specification of a two-party protocol. The party $P_1$ provides an integer $a$ and the party $P_2$ provides an integer $b$. The program computes securely the following functionality: if $a < b$, the protocol outputs 1 to both parties, otherwise it outputs 0.

If we compile the previous program, we obtain the following output on the terminal:

```plaintext
Program requires at most:
           1 integer inputs from player 0
           1 integer inputs from player 1
           1 strict edabits of length 41
           1 strict edabits of length 64
         120 bit triples
           1 integer dabits
          19 virtual machine rounds
```

Note that the compilation process shows that the protocol requires the production of an integer daBit in the pre-processing phase. 

In the previous example, the need for a daBit automatically comes from the compilation process, however, I am more interested in showing how to tell the MP-SPDZ framework that we want to make a conversion using daBits explicitly. To do this, I will use the following source code as an example:

```python
# This line tells the compiler that we allow the use of daBits/edaBits.
program.use_edabit(True)

# Here, we are saying that the bit strings that we are manipulating in the
# binary domain have just one bit.
sb = sbits.get_type(1)

# Parties' inputs
a = sb.get_input_from(0)
b = sb.get_input_from(1)
c = sint.get_input_from(3)

# Computation of the XOR
xor = a.bit_xor(b)

# Explicit domain conversion. We are converting xor from the binary domain into
# the arithmetic domain.
xor_int = sint(xor)

# Secure multiplication
result = xor_int * c

# Delivering the results to all parties.
print_ln("%s", result.reveal())
```

First, let me explain the functionality presented above. The previous source code is the specification of a three-party protocol in which $P_1$ and $P_2$ provide two bits $a, b \in \mathbb{Z}\_2$ respectively, and $P_3$ provides a value $c \in \mathbb{Z}\_q$ for a given fixed prime $q$. It is important to mention that $c$ could belong to any arithmetic domain, but I choose $Z_q$ for this discussion. In this example, the parties agree on computing the following function securely:

$$
f(a, b, c) =
\begin{cases}
  c, & \text{if $a \neq b$} \\\\
  0, & \text{if $a = b$}
\end{cases}
$$

In the protocol, the parties first compute securely $\llbracket x \rrbracket_2 = \llbracket a \rrbracket_2 \oplus \llbracket b \rrbracket_2$ in the binary domain. Then, using daBits, the parties transform $\llbracket x \rrbracket_2$ into $\llbracket x \rrbracket_q$. Finally, the parties compute securely $\llbracket r \rrbracket_q = \llbracket x \rrbracket_q \cdot \llbracket c \rrbracket_q$ and reveal the value $r$.

In the source code, each line is explained line-by-line. The relevant instruction is `xor_int = sint(xor)`. Here, I am telling the compiler that I need an explicit conversion of the variable `xor` from the binary domain to the arithmetic domain. Once this instruction is computed, the parties will have shares of the value $a \oplus b$ but in the arithmetic domain. The conversion allows the secure multiplication between $a \oplus b$ and $c$ in the arithmetic domain.

As a closing remark, the examples presented here can be a bit easy and somewhat artificial. However, these concepts and techniques have real uses in more complex protocols. As an example (and a bit of self-promotion), the work of Vanegas et al. (2023) presents a two-party protocol that computes the edit distance securely using a mixed-circuit computation. First, the protocol in that paper computes some comparisons in the binary domain, and then the result of such comparisons is transformed into the arithmetic domain to finally compute the edit distance between two strings in a finite alphabet. To make the transition between the arithmetic domain to the binary domain, the authors use the techniques presented here.

## References

- Aly, A., Orsini, E., Rotaru, D., Smart, N. P., & Wood, T. (2019). Zaphod: Efficiently Combining LSSS and Garbled Circuits in SCALE. Proceedings of the 7th ACM Workshop on Encrypted Computing & Applied Homomorphic Cryptography, 33–44. Presented at the London, United Kingdom. doi:10.1145/3338469.3358943.
- Damgård, I., Escudero, D., Frederiksen, T., Keller, M., Scholl, P., & Volgushev, N. (2019). New Primitives for Actively-Secure MPC over Rings with Applications to Private Machine Learning. 2019 IEEE Symposium on Security and Privacy (SP), 1102–1120. doi:10.1109/SP.2019.00078.
- Keller, M. (2020). MP-SPDZ: A Versatile Framework for Multi-Party Computation. Proceedings of the 2020 ACM SIGSAC Conference on Computer and Communications Security. doi:10.1145/3372297.3417872.
- Rotaru, D., & Wood, T. (2019). MArBled Circuits: Mixing Arithmetic and Boolean Circuits with Active Security. In F. Hao, S. Ruj, & S. Sen Gupta (Eds.), Progress in Cryptology -- INDOCRYPT 2019 (pp. 227–249). Cham: Springer International Publishing.
- Vanegas, H., Cabarcas, D., & Aranha, D. F. (2023). Privacy-Preserving Edit Distance Computation Using Secret-Sharing Two-Party Computation. Progress in Cryptology – LATINCRYPT 2023: 8th International Conference on Cryptology and Information Security in Latin America, LATINCRYPT 2023, Quito, Ecuador, October 3–6, 2023, Proceedings, 67–86. Presented at the Quito, Ecuador. doi:10.1007/978-3-031-44469-2_4.
