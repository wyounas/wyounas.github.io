---
layout: post
title:  "Solving the LinkedIn Queens Puzzle with a SAT Solver and Two Model Checkers"
date: 2025-8-18
categories: model-checking, puzzles
---


<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/first.png" />

LinkedIn Queens is a puzzle played on an `n × n` grid with the following rules:

- The grid is divided into `n` colored regions.
- Each row, column, and colored region must contain exactly one queen.
- No two queens may be adjacent, including diagonally. However, diagonals at a distance are allowed.

In this post, we aim to find all solutions, not just one, to the puzzle. We aim to find all solutions using a Python-based SAT solver and two model checkers, namely Spin and Fizzbee. To keep things simple, we’ll solve the puzzle for a 4x4 grid using the SAT Solver, but we will solve it both for 4x4 and 9x9 using model checkers. 

There are different ways to solve this problem. My goal here is to explore how various computational approaches can tackle the same challenge. I believe that model checkers, for instance, aren't just powerful tools for ensuring correctness, but also a great way to learn about nondeterminism. There's also a touch of elegance in how SAT solvers can sometimes reformulate complex problems.

## Solving the Puzzle using a SAT Solver

I first came across this puzzle in a nice [blog post](https://buttondown.com/hillelwayne/archive/solving-linkedin-queens-with-smt/) by Hillel Wayne, whose blog I enjoy reading, where he solved it using an SMT solver called Z3. 


In the past, I had used a Python-based SAT solver called PyEDA [1], and I thought it would be interesting to use it to find all solutions to the LinkedIn Queens puzzle. Let’s solve it for a 4x4 grid though the code can be extended to solve it for a 9x9 grid. The code uses Python 3.13.1 and PyEDA 0.29.0. All the code in this post is linked in relevant sections, and the repository containing all the code is also linked in the references at the end.

To get started with PyEDA, we represent the 4×4 grid as a two-dimensional bit vector:
```
X = exprvars('x', 4, 4)
```

To represent colored regions, we assign each region a number. For example, cells marked with 1 belong to the red region, 2 to green, 3 to yellow, and 4 to pink.

Consider the following layout of four regions in a 4×4 grid, where each row is assigned a distinct color:
```
1 1 1 1
2 2 2 2
3 3 3 3
4 4 4 4
```
We can map this layout using a Python dictionary as follows:
```
 regions = {
        1: [(0, 0), (0, 1), (0, 2), (0, 3)],  # Region 1: cells 1-4 (row 0)
        2: [(1, 0), (1, 1), (1, 2), (1, 3)],  # Region 2: row 1
        3: [(2, 0), (2, 1), (2, 2), (2, 3)],  # Region 3: row 2
        4: [(3, 0), (3, 1), (3, 2), (3, 3)]   # Region 4: row 3
    }
```
   
Next, we encode the constraints that reflect the rules of the puzzle. 

The first constraint ensures that each row contains exactly one queen. To add the constraint, we make use of the OneHot() method in PyEDA which provides a guarantee that only one of the variables in a boolean formula is true at any given moment. Here’s how we express this constraint for each row:
```
 for r in range(n):
        row_vars = [X[r,c] for c in range(n)]
        constraints.append(OneHot(*row_vars))
```

In the same manner, we add a constraint to ensure that each column contains exactly one queen, satisfying another rule of the puzzle:
```
for c in range(n):
        col_vars = [X[r,c] for r in range(n)]
        constraints.append(OneHot(*col_vars))
```

Finally, we need to add the constraint that enforces the adjacent diagonal rule. Since the earlier constraints already prevent queens from being placed in the same row or column (i.e., non-diagonal adjacency), we only need to ensure that no two queens are placed in adjacent diagonal cells. So in a `4x4` grid, if there is a queen at (1,1), then we can't place a queen at the left diagonal (2,0) and right diagonal (2,2) as shown in the illustration below.

```
 .  .  .  . 
 .  Q  .  . 
 X  .  X  . 
 .  .  .  . 

```

For each cell on the board, we want to make sure that if there's a queen in the cell, there cannot be queens on the diagonal cells directly below it. What about the upper diagonals? We don’t have to check for them. Because, when we process row 0, we prevent conflicts with row 1. And when row 1 is processed, those constraints already exist. This way, we avoid duplicate constraints. 

Here’s the code that implements this. It includes a few small optimizations: for example, we skip left-diagonal checks in column 0, since there are no cells to the left, and we stop at the second-to-last row, because by that point we’ve already covered all necessary diagonal constraints.
```
for r in range(n-1):  # Stop before last row
        for c in range(n):
            # Below-left diagonal
            if c > 0:
                # A constraint like ~(X[1,1] & X[2,0]) means you can't have 
                # queens at both (1,1) and (2,0) because ~(a & b) means both
                # 'a' and 'b' can't be both true at the same time. 
                constraints.append(~(X[r,c] & X[r+1, c-1]))
            # Below-right diagonal
            if c < n-1:
                # A constraint like ~(X[1,1] & X[2,2]) means you can't have 
                # queens at both (1,1) and (2,2)
                constraints.append(~(X[r,c] & X[r+1, c+1]))
```

That’s pretty much all we need to do to get not just one, but all possible solutions to the puzzle. We then print the total number of solutions, which turns out to be two.

Below are two valid solutions for the above board with specific regions. Here is the [complete code](https://github.com/wyounas/linkedin_queens/blob/main/queens.py) that generated these two solutions.

Solution 1 - Queens at cells: [3, 5, 12, 14]:
```
 .  .  Q  . 
 Q  .  .  . 
 .  .  .  Q 
 .  Q  .  . 

```

Solution 2 - Queens at cells: [2, 8, 9, 15]:
```
 .  Q  .  . 
 .  .  .  Q 
 Q  .  .  . 
 .  .  Q  . 
```

Now let’s try to solve the puzzle using a model checker, again with the goal of finding all possible solutions, not just one. We'll start with Spin, which uses Promela as its modeling language.

## Solving the Puzzle Using Spin 

Model checkers are tools designed to verify whether a system satisfies a given correctness specification by exploring all possible computations of the system. For concurrent and nondeterministic programs, this involves systematically exploring all possible interleavings and using backtracking to exhaustively explore the state space. The state space of a program is the set of all possible states that can occur during a computation. 

We're not going to use model checking in the traditional sense in that we're not going to check safety or liveness properties. But we'll use Spin for its excellent support for nondeterminism, which is a central idea in how we solve this puzzle. I also believe Spin is an excellent tool for learning about nondeterminism.

The verification process in Spin typically involves three steps:

- Write the model in Promela and save it in a `.pml` file.
- Generate a verifier. One can generate it by running `spin -a model.pml`. This generates a `pan.c` file (name `pan` is historically derived from ‘protocol analyzer’). Then compile it with a C compiler (e.g., `gcc -o pan pan.c`).
- Run the verifier. One can run it using the shell command `./pan`. It reports whether all computations are correct or there exists a computation that contains an error. If an error is found, the verifier produces a trail file which can be used to reconstruct the erroneous computation. We'll make use of trail files shortly, to extract valid puzzle solutions. More on this in a bit.


Instead of  `spin -a model.pml`, we can also run `spin -run model.pml` to generate the verifier and compile it in one go. 

Before diving into the solution, let’s briefly discuss two Spin/Promela concepts that we'll use to solve the puzzle. 

The first concept is a guard expression that blocks the process when it evaluates to false. Blocking here means that Spin will not execute the next statement from this process. The following statement will block the above process when ‘x’ equals 2:
```
! (x == 2);
```

The statement above allows the process to make progress only if the condition evaluates to true. The ! operator is a logical negation, so the above guard expression is true when x is not equal to 2. In other words, the process will proceed only when x is not 2.

The second concept is how nondeterminism and error search work in Spin. Consider the following small Promela program (All Spin code in this post was written using Spin version 6.5.2):

<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/second.png" />

We wrap our code in a process-like executable unit using the `proctype` keyword in Promela. When we prefix a `proctype` definition with the `active` keyword, we’re telling the model checker to automatically start that process when Spin begins execution. 

The `if` statement above is how nondeterministic choice is expressed in Promela. In this case, the `if` block allows Promela to choose a value for `x` nondeterministically. After this `if` statement is executed, `x` will have one of the values: 1, 2, 3, 4, or 5.

To further understand how this nondeterminism will play out, below is an automata that represents the possible transitions for this code (generated using a tool called jSpin [2]):

<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/three.png" />

The numbers inside the circles represent line numbers. As you can see, there isn’t just one computation path from start to finish, there are five possible computations, depending on which value is assigned to `x`.

At the end of the above program, we have an assert that checks whether `x == 1`. Will this assertion pass? It will fail, because Spin finds a counterexample. 

How does it find a counterexample? The assert causes the model checker to backtrack and explore all possible counterexamples from all computations. It tries to find computations where `x` is not set to 1. For example, from five computations shown in the above figure, it will find paths where `x` is set to 3, 4, or 5—but not 2, because that choice was blocked earlier using the guard expression.

Here’s a truncated version of the assertion violation report you would see when you run your model (assuming the above code is saved in a file named atest.pml):

```
$ spin -run atest.pml  
$ ./pan -E         
pan:1: assertion violated (x==1) (at depth 1)
pan: wrote atest.pml.trail
```

The -E switch tells the verifier to ignore computations that end in invalid status which could be caused by blocked computations.
As indicated in the above output, this generates a trail file. We can replay it with Spin’s `-t -p` switches, inspect the output, and see how the computation led to the error. The output below, truncated for clarity, shows the sequence of steps that led to the assertion failure.

```
$ spin -t -p atest.pml

  1:    proc  0 (P:1) atest.pml:7 (state 3)     [x = 3]
  2:    proc  0 (P:1) atest.pml:12 (state 8)    [(!((x==2)))]
spin: atest.pml:13, Error: assertion violated
```

The output above shows that the assertion failed in the computation where `x` was set to 3.  But there are other counterexamples that would also cause the assertion to fail, for example, when `x` is set to 4 or 5. So how do we find all these counterexamples? We can do that using the following command, which enables exploration of all computations:

```
./pan -E -c0 -e    
pan:1: assertion violated (x==1) (at depth 1)
pan: wrote atest.pml1.trail
pan: wrote atest.pml2.trail
pan: wrote atest.pml3.trail
```

For each counterexample, Spin generates a separate trail file (e.g., attest.pml1.trail, etc). We can replay these trail files to reconstruct the exact sequence of steps that led to the assertion failure. For example, let’s inspect the contents of atest.pml3.trail to see another such erroneous computation:

```
spin -p -t3 atest.pml
  1:    proc  0 (P:1) atest.pml:9 (state 5)     [x = 5]
  2:    proc  0 (P:1) atest.pml:12 (state 8)    [(!((x==2)))]
spin: atest.pml:13, Error: assertion violated
```

We see that when `x` was set to 5, it refuted our assertion claim. 

The key takeaway is this: we nondeterministically set `x`, wrote an assertion, and let the model checker backtrack and explore all possible computations involving the different values `x` could take and the model checker found all computations that violated the assertion and reported them as counterexamples.

This ability of the model checker (Spin in this case) to take code with nondeterminism and systematically backtrack and explore all possible computations in order to find a counterexample to an assertion is central to how we’ll find all solutions to the puzzle. Ben-Ari’s book [3] introduced me to these ideas.

So how do we find those valid combinations that are solutions to LinkedIn Queens puzzle?

Figure 1 helped me come up with a solution. In that figure, we chose a value for `x` nondeterministically from five options. The model checker then explored all five possible computations, one for each choice. Similarly, if we can nondeterministically choose a cell from each region, the model checker will evaluate all possible combinations of four cells, with one cell chosen from each region.

To visualize this, I found it helpful to imagine a tree-like structure, as shown in Figure 2 below (If the image appears unclear, please consider opening it in a new tab.). It’s of course not ideal, but it helped me think through the problem. For both Figure 2 and Figure 3 below, we use the same board and colored region configuration as in the Python code above—where 1s represent region colored red, 2s represent region colored green, 3s represent region in yellow, and 4s represent region in pink. The board layout looks like this:
```
1 1 1 1
2 2 2 2
3 3 3 3
4 4 4 4
```

In this configuration:
- Region 1 includes cells 1, 2, 3, 4
- Region 2 includes cells 5, 6, 7, 8
- Region 3 includes cells 9, 10, 11, 12
- Region 4 includes cells 13, 14, 15, 16

We could now imagine a tree with four levels. At Level 0, you see all the cells from Region 1. From each of those cells, there are branches leading to every cell in Region 2.
In the figure, in Region 1, we show a branch going out from cell 1 just to save space—but you can imagine similar branches coming from cells 2, 3, and 4 as well. And the same for all cells in level 1 and 2. 

<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/four.png" />

Now let’s see what’s going on in Figure 2. A queen is first placed in cell 1, and then the search moves on to Region 2, where cells 5 and 6 are ignored because of the constraints. It places the queen in cell 7. The board at this point is shown on the right. When it moves to Region 3, it hits a dead-end—there is no cell available where the queen can be placed without violating the constraints. 

At that point, we could imagine Spin backtracking. Let’s look at the following illustration to see what happens next. 


<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/five.png" />

After hitting a dead-end, it backtracks to Region 1 (visualized as the red arrow going from 7(Q) back to 2(Q)). It places the queen in cell 2 in region 1 (shown as 2(Q), colored in green). After successfully placing a queen there, it moves on to region 2, where it rejects cell 6 due to a column conflict, and cells 5 and 7 due to diagonal clashes. It then selects cell 8. In region 3, given all constraints, it places the queen in cell 9, and finally, in region 4, respecting constraints, it finds a valid position at cell 15, thus completing one valid solution in compliance with the row, column, and region constraints (the solved board is shown at the right).

Now let’s begin working toward a solution in Spin, with the goal of finding all possible solutions to the puzzle. We’ll solve it first for a 4x4 grid as it’s simple, and then the same code, with minor changes, would be used to solve it for a 9x9 grid. 

We use the same board and colored regions configuration as given above.

We encode colored region of 1s as:
```
inline ChooseRegion1() {
    if
    :: cell = 1   
    :: cell = 2    
    :: cell = 3   
    :: cell = 4  
    fi
}
```

And region 2 as:

```
inline ChooseRegion2() {
    if
    :: cell = 5    
    :: cell = 6   
    :: cell = 7    
    :: cell = 8   
    fi
}
```

And so on. Why make the model checker choose cells nondeterministically? We can connect this to Figure 2 and 3 and the explanation that followed. By organizing regions and their cells as shown in these figures you can see how this structure helps the model checker systematically explore all combinations—by branching through every possible choice at each level.

We also define a few variables at the top of the model:

```
#define N 4
byte result[N];    
bool rows[N];      
bool cols[N];           
bool regions[N];
bool diagonals[N*N]; 
```

We’ll number the colored regions from 1 to 4, and the cells from 1 to 8 (starting from the top-left cell, counting left to right across each row, and ending at the bottom-right cell).

The `result` array stores the final solution. Each `result[region-1]` holds the cell where a queen is placed in that region. For example, `result[0] = 2` means the queen is placed in cell 2 in region 1.

The array `rows` keeps track of whether a queen has already been placed in a given row. For example, `rows[i]` is true if there is a queen in row `i`. Similarly, `cols` keeps track of whether a queen has been placed in a given column. 

The array `diagonals` stores information about the diagonal constraints. 

We process the board region by region and once we’ve processed a region we store it in an `regions` array. 

The general outline of what happens in the program is as follows. As the program processes the board region by region iteratively, from region 1 to 4, it performs these steps in sequence (we’ll go over each in some detail shortly):

- Iterate through all regions and do the following in each iteration:
    - Choose cells for a region nondeterministically.
    - Checks whether a queen is already placed in the current row and column. If we’ve placed the queen in the current row or column already, we block the process.
    - Ensures that we’re not placing a queen on an adjacent diagonal. If we are, the process is blocked.
    - Mark the row and column as processed.
    - Places the queen and records its position in the `result` array.
    - Mark the queen’s adjacent diagonals to prevent placing another queen in those cells later.
    - Print the results.

We use `assert(false)` at the end to make the model checker backtrack and explore all computations that successfully exit the loop.

Let’s quickly go over these one by one. 

We start a loop to process the board region by region, and the first step inside the loop is to choose region cells nondeterministically using the following code:
``` 
if
        :: region == 1 -> ChooseRegion1()
        :: region == 2 -> ChooseRegion2()
        :: region == 3 -> ChooseRegion3()
        :: region == 4 -> ChooseRegion4()
        fi

```

The program then checks and blocks if a queen has already been placed in the chosen row. This is done using a guard expression:
```
!rows[row];  
```

As we discussed earlier, the execution continues if above is true, meaning the row is still available. It’s important to note that, because of these blocking statements, many computations will be blocked, but some won’t. To find valid solutions to the puzzle, we’ll make use of the assertion claim. More on this in a bit.

After checking the row, we similarly check and block the placement if a queen has already been placed in the same column. We also block the cell if it lies on an adjacent diagonal to an existing queen.

Once a valid cell is found, we mark the row and column as occupied and we store the queen’s position in the result array:

```
rows[row] = true;
cols[col] = true;      
result[region-1] = cell;
```

Finally, we update the `diagonals` array to record the adjacent lower diagonal cells of the queen we just placed. These will be used later to ensure that no new queen is placed on an adjacent diagonal. This step prepares the model to block any future placements that would violate the diagonal adjacency constraint.

At this point, we’ve finished processing the region, so we move on to the next by restarting the loop. This process repeats until all regions have been processed.

As you can see, near the end, we’ve not only printed the `result` array, but also, we’ve added this line:
```
assert(false); 
```

This causes the model checker to backtrack and search for computations that successfully exit the loop, meaning computations that have placed queens on the board according to the rules. Any such computation is not just a counterexample to our assertion claim, but also a valid solution to the puzzle. [Here is the code](https://github.com/wyounas/linkedin_queens/blob/main/queenfourbyfour.pml) for 4x4 grid. 

Now to find a solution, we run the model with `spin -run queens.pml` and then run the verifier `./pan -E`.
You’ll see an assertion violation, along with a message indicating that a trail file has been written. Replay the trail file with command:
```
spin -p -t queens.pml
```

The above will generate output that includes the contents of the `result` array, our solution to the puzzle. Somewhere near the end of the generated output, you’ll see the `result` array printed:

```
result[0] = 2
result[1] = 8
result[2] = 9
result[3] = 15
```

If we plot the above values from the result array onto a board, we get a valid solution to the LinkedIn Queens puzzle:
```
 .  Q  .  . 
 .  .  .  Q 
 Q  .  .  . 
 .  .  Q  . 
```

But this gives us just one solution to the puzzle. How can we find all solutions, ideally without adding any more code to our existing model? As we did earlier, by running `./pan -E -c0 -e`. You’ll get two solutions to the puzzle in respective trail files i.e., `queens.pml1.trail` and `queens.pml2.trail`. Here is the second solution:

```
result[0] = 3
result[1] = 5
result[2] = 12
result[3] = 14

 .  .  Q  . 
 Q  .  .  . 
 .  .  .  Q 
 .  Q  .  . 
```

Without changing much code, we can extend these ideas to a 9x9 grid. Here is a LinkedIn Queens puzzle by LinkedIn on a day on July 2025:

<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/six.png" />

[Here is the complete code](https://github.com/wyounas/linkedin_queens/blob/main/queenninebynine.pml) that gives us the solution to this puzzle. When I run it, I find the solution:
```
result[0] = 46
result[1] = 11
result[2] = 6
result[3] = 26
result[4] = 39
result[5] = 32
result[6] = 63
result[7] = 76
result[8] = 70
```


<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/seven.png" />

I took the original image and, based on the values in the `result` array, placed queens (marked as "Q" above) in the image. It looks like a valid solution to the puzzle that satisfies all the constraints. (Please ignore the small ‘x’ in the image above, it was just a placeholder in the original image.)

Just for fun and curiosity, I wanted to find all solutions to the puzzle without the region constraint, that is, to find all valid queen placements where exactly one queen is placed in each row and column, and no two queens are adjacent, including diagonally.

[This code](https://github.com/wyounas/linkedin_queens/blob/main/queens_wo_region.pml) does that, and it reports 5242 solutions to the puzzle when we drop the region constraint for a `8 x 8` grid (please note that if you run this, it'll generate as many trail files as there are solutions).

In the end, I’d say that Spin’s support for nondeterminism is quite powerful and it can be leveraged to solve certain kinds of problems in an elegant way.

## Solving the puzzle with Fizzbee


There’s another promising model checker on the horizon called [Fizzbee](https://fizzbee.io/). Its syntax closely resembles Python, which makes it more approachable, especially for programmers who prefer Python-like semantics.

To find a solution using Fizzbee, I applied similar ideas to those I used with Promela and Spin.

Just as we wrap the code we want to explore in a `proctype` in Promela to model process-like behavior, in Fizzbee we use an `action` block to define such executable units.

We begin by writing the `Init` action, which is executed once at the start. It is responsible for setting up the system state and initializing all relevant variables.

The overall logic is the same as in the Promela version: we process one region at a time. Before placing a queen, we check whether the target cell is on an adjacent diagonal to an already-placed queen. We also use guard conditions via the `require` keyword to block the action if a queen has already been placed in the row or column we’re trying to use.

I wasn’t able to fully explore Fizzbee’s support for nondeterminism, but from what I saw, it appears to not have constructs that look and behave very similarly to those in Spin. (It’s possible I may have overlooked something.) So to find a solution, we add an assertion that checks for an invalid board configuration. Why an invalid board configuration? Because the model checker will then try to find a counterexample, and that counterexample will be a valid solution to the puzzle.

To check for an invalid board, we use an assertion that looks for three things: if there is more than one queen in any row, more than one queen in any column, or if any queens are touching diagonally, or if more than one queen in the region. Here is the full code.

When I run this [code](https://github.com/wyounas/linkedin_queens/blob/main/queens.fizz) for the above 9x9 LinkedIn Queens puzzle, Fizzbee returns one counterexample. And the counterexample corresponds to a valid LinkedIn Queens solution.

Below is the resulting 9x9 board, where 1 represents a queen placed in that cell:

```
[0, 0, 0, 0, 0, 1, 0, 0, 0]
[0, 1, 0, 0, 0, 0, 0, 0, 0]
[0, 0, 0, 0, 0, 0, 0, 1, 0]
[0, 0, 0, 0, 1, 0, 0, 0, 0]
[0, 0, 1, 0, 0, 0, 0, 0, 0]
[1, 0, 0, 0, 0, 0, 0, 0, 0]
[0, 0, 0, 0, 0, 0, 0, 0, 1]
[0, 0, 0, 0, 0, 0, 1, 0, 0]
[0, 0, 0, 1, 0, 0, 0, 0, 0]
```

So far, I’ve only been able to get Fizzbee to output a single counterexample as a solution even when I use a board that has multiple solutions. Currently, Fizzbee doesn’t list all counterexamples. However, JP (the creator of Fizzbee, who is always helpful) showed me an alternative approach. He suggested printing the board inside the assertion code at the end and returning true instead of false.

_Please keep in mind that I’m only human, and there’s a good chance this post contains errors—even though I’ve done my best to avoid them. If you notice anything off, I’d appreciate a correction. Please feel free to [send me an email](mailto:waqas.younas@gmail.com)._


## References

1. PyEda: https://pyeda.readthedocs.io/en/latest/overview.html 
2. jSpin: https://github.com/motib/jspin
3. [Ben-Ari, M. (2008). Principles of the Spin model checker. Springer.](https://link.springer.com/book/10.1007/978-1-84628-770-1)
4. Repository containing all code in this post: https://github.com/wyounas/linkedin_queens