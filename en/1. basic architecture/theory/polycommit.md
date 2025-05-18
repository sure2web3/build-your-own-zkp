# Polycommit Module
* Polynomial Commitment Scheme: A polynomial commitment scheme allows a prover to commit to a polynomial and later prove evaluations of the polynomial at specific points. The commitment hides the polynomial coefficients, while the proof allows the verifier to check the correctness of the evaluation without learning the coefficients. The [Halo] paper describes a specific polynomial commitment scheme that is used in this implementation.
* Fiat-Shamir Transformation: The Fiat-Shamir transformation is a technique for converting an interactive zero-knowledge proof into a non-interactive one. It uses a hash function (represented by the sponge function in this code) to generate challenges based on the prover's commitments. The challenges are used to ensure the soundness of the non-interactive proof.

## Polynomial Commitment Scheme

## Fiat-Shamir Transformation