What is Tensor Comprehensions?
==============================

Tensor Comprehensions(TC) is a notation based on generalized Einstein notation
for computing on multi-dimensional arrays. TC greatly simplifies ML framework
implementations by providing a concise and powerful syntax which can be efficiently
translated to high-performance computation kernels, automatically.

Example of using TC with framework
----------------------------------

TC is supported both in Python and C++ and we also provide lightweight integration
with PyTorch/Caffe2 frameworks.

An example of how using TC in PyTorch looks like:

.. code-block:: python

    import tensor_comprehensions as tc
    import torch
    lang = """
    def matmul(float(M,N) A, float(N,K) B) -> (output) {
      output(i, j) +=! A(i, kk) * B(kk, j)
    }
    """
    matmul = tc.define(lang, name="matmul")
    mat1, mat2 = torch.randn(3, 4).cuda(), torch.randn(4, 5).cuda()
    out = matmul(mat1, mat2)


For more details on how to use TC with PyTorch, see :ref:`tc_with_pytorch`.

More generally the only requirement to integrate TC into a workflow is to use a
simple tensor library with a few basic functionalities. For more details, see
:ref:`integrating_ml_frameworks`.

.. _tc_einstein_notation:

Tensor Comprehension Notation
-----------------------------
TC borrow three ideas from Einstein notation that make expressions concise:

1. loop index variables are defined implicitly by using them in an expression and their range is aggressively inferred based on what they index,
2. indices that appear on the right of an expression but not on the left are assumed to be reduction dimensions,
3. the evaluation order of points in the iteration space does not affect the output.

Let's start with a simple example is a matrix vector product:

.. code::

    def mv(float(R,C) A, float(C) B) -> (o) {
      o(i) += A(i,j) * B(j)
    }

:code:`A` and :code:`B` are input tensors. :code:`o` is an output tensor.
The statement :code:`o(i) += A(i,j) * B(j)` introduces two index variables :code:`i` and :code:`j`.
Their range is inferred by their use indexing :code:`A` and :code:`B`. :code:`i = [0,R)`, :code:`j = [0,C)`.
Because :code:`j` only appears on the right side,
stores into :code:`o` will reduce over :code:`j` with the reduction specified for the loop.
Reductions can occur across multiple variables, but they all share the same kind of associative reduction (e.g. :code:`+=`)
to maintain invariant (3). :code:`mv` computes the same thing as this C++ loop:

.. code::

    for(int i = 0; i < R; i++) {
      o(i) = 0.0f;
      for(int j = 0; j < C; j++) {
        o(i) += A(i,j) * B(j);
      }
    }

The loop order :code:`[i,j]` here is arbitrarily chosen because the computed value of a TC is always independent of the loop order.

Examples of TC
--------------

We provide a few basic examples.

Simple matrix-vector
^^^^^^^^^^^^^^^^^^^^

.. code::

    def mv(float(R,C) A, float(C) B) -> (o) {
      o(i) += A(i,j) * B(j)
    }

Simple 2-D convolution (no stride, no padding)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code::

    def conv(float(B,IP,H,W) input, float(OP,IP,KH,KW) weight) -> (output) {
      output(b, op, h, w) += input(b, ip, h + kh, w + kw) * weight(op, ip, kh, kw)
    }

Simple 2D max pooling
^^^^^^^^^^^^^^^^^^^^^^

Note the similarity with a convolution with a "select"-style kernel:

.. code::

    def maxpool2x2(float(B,C,H,W) input) -> (output) {
      output(b,c,i,j) max= input(b,c,2*i + kw, 2*j + kh)
        where kw in 0:2, kh in 0:2
    }
