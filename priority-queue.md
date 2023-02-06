# Priority Queue Chisel Library

## 11/10/2022

Sungwoong:

• FIFO and testbench look good
    ◦ For the FIFO, can you avoid tracking the number of elements in the FIFO altogether? Can you reduce thes number of bits of state in the FIFO?
    ◦ Hint: see the Wikipedia page for FIFO
    ◦ https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)
    ◦ See the “Status Flags” section
• Priority Queue in Chisel
    ◦ https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9730477&tag=1
    ◦ TODO Vighnesh: attach more links here + IO spec + our initial prototype
    ◦ https://github.com/joey0320/chisel-priorityqueue/blob/for-huffman/src/main/scala/shift_register_pq.scala
    ◦ https://github.com/joey0320/chisel-priorityqueue
    ◦ https://ieeexplore.ieee.org/document/895938
    ◦ http://fmdb.cs.ucla.edu/Treports/140013.pdf
