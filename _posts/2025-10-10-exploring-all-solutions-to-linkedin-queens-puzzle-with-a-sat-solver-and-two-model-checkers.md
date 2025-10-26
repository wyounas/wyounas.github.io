---
layout: post
title:  "Exploring All Solutions to LinkedIn Queens Puzzle with a SAT Solver and Two Model Checkers"
date: 2025-10-18
categories: ["model-checking", "puzzles"]
---


<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/first.png" />

LinkedIn Queens is a puzzle played on an `n × n` grid with the following rules:

- The grid is divided into _`n`_ colored regions.
- Each row, column, and colored region must contain exactly one queen.
- No two queens may be adjacent, including diagonally. However, diagonals at a distance are allowed.

In this post, we find all solutions, not just one, using a Python‑based SAT solver and two model checkers (Spin and FizzBee). To keep things simple, we’ll solve the puzzle for a `4x4` grid using the SAT solver, but we will solve it both for `4x4` and `9x9` using model checkers. 

The goal here is to explore how various computational approaches can tackle the same problem. For one, I believe there is a touch of elegance in how SAT solvers can sometimes reformulate complex problems. I also believe that model checkers, for instance, aren't just powerful tools for ensuring correctness, but also a great way to learn about nondeterminism. 

All the code in this post is linked in relevant sections, and the repository containing all the code is also linked in the references at the end.

## Solving the Puzzle using a SAT Solver

I first came across this puzzle in a nice [blog post](https://buttondown.com/hillelwayne/archive/solving-linkedin-queens-with-smt/) by Hillel Wayne, whose blog I enjoy reading, where he solved it using an SMT solver called Z3. 


In the past, I had used a Python-based SAT solver called PyEDA [1], and I thought it would be interesting to use it to find all solutions to the LinkedIn Queens puzzle. Let’s solve it for a `4x4` grid though the code extends to `9x9` grid. The code uses Python 3.13.1 and PyEDA 0.29.0. 

To get started with PyEDA, we represent the 4×4 grid as a two‑dimensional array of Boolean variables:
```python
X = exprvars('x', 4, 4)
```

To represent colored regions, we assign each region a number. For example, cells marked with 1 belong to the red region, 2 to green, 3 to yellow, and 4 to pink.

Consider the following layout of four regions in a `4×4` grid, where each row is assigned a distinct color:
```
1 1 1 1
2 2 2 2
3 3 3 3
4 4 4 4
```
We can map this layout using a Python dictionary as follows:
```python

regions = {
        1: [(0, 0), (0, 1), (0, 2), (0, 3)],  # Region 1: cells 1-4 (row 0)
        2: [(1, 0), (1, 1), (1, 2), (1, 3)],  # Region 2: row 1
        3: [(2, 0), (2, 1), (2, 2), (2, 3)],  # Region 3: row 2
        4: [(3, 0), (3, 1), (3, 2), (3, 3)]   # Region 4: row 3
    }
```
   
Next, we encode the constraints that reflect the rules of the puzzle. 

The first constraint ensures that each row contains exactly one queen. To add the constraint, we make use of the OneHot() function in PyEDA which enforces that exactly one variable in the list is true. Here’s how we express this constraint for each row:
```python
for r in range(n):
    row_vars = [X[r,c] for c in range(n)]
    constraints.append(OneHot(*row_vars))
```

In the same manner, we add a constraint to ensure that each column contains exactly one queen, satisfying another rule of the puzzle:
```python
for c in range(n):
    col_vars = [X[r,c] for r in range(n)]
    constraints.append(OneHot(*col_vars))
```

Similarly, for each colored region we also use `OneHot` to enforce one‑queen‑per‑region rule.

Finally, we need to add the constraint that enforces the adjacent diagonal rule. Because rows and columns already have exactly one queen, horizontal or vertical neighbors can’t happen. The only remaining local conflict is immediate diagonal adjacency. For example, a queen at (1, 1) forbids (0, 0), (0, 2), (2, 0), and (2, 2) as shown in the illustration below.

```
 X  .  X  . 
 .  Q  .  . 
 X  .  X  . 
 .  .  .  . 

```

We only add constraints for each cell and its two lower diagonals (r+1, c±1). That way, every adjacent‑diagonal pair appears exactly once, when we visit its higher row. Adding the symmetric “upper” pairs would just duplicate constraints.

Here is the code that implements this. It includes a few small optimizations: for example, we skip left-diagonal checks in column 0, since there are no cells to the left, and we stop at the second-to-last row, because all necessary diagonal pairs have already been covered.
```python
for r in range(n-1):  # Stop before last row
        for c in range(n):
            # Below-left diagonal
            if c > 0:
                # A constraint like ~(X[1,1] & X[2,0]) means you can't have 
                # queens at both (1,1) and (2,0) because ~(a & b) implies both
                # 'a' and 'b' can't be both true at the same time. 
                constraints.append(~(X[r,c] & X[r+1, c-1]))
            # Below-right diagonal
            if c < n-1:
                # A constraint like ~(X[1,1] & X[2,2]) means you can't have 
                # queens at both (1,1) and (2,2)
                constraints.append(~(X[r,c] & X[r+1, c+1]))
```

That’s pretty much all we need to do to get not just one, but all possible solutions to the puzzle. We combine all the constraints and call PyEDA’s `satisfy_all()` to enumerate every valid board.

Below are two valid solutions for the above board with specific regions. Here is the [complete code](https://github.com/wyounas/linkedin_queens/blob/main/queens.py) that generated these two solutions.

Solution 1 - Queens at cells: [3, 5, 12, 14] (Cells are numbered from 1 to 16 (top‑left is 1, bottom‑right is 16)):
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

We are not going to use model checking in the traditional sense in that we are not going to check safety or liveness properties. Instead, we’ll use Spin for its excellent support for nondeterminism, which we also rely on to solve this puzzle. More on this below, with an example.

Before we turn to that example, let’s quickly review Spin’s verification process. Spin analyzes the correctness of a system through verification, which typically involves three main steps:

- Write the model in Promela and save it in a `.pml` file.
- Generate a verifier. One can generate it by running _`spin -a model.pml`_. This generates a `pan.c` file (name `pan` is historically derived from ‘protocol analyzer’). Then compile it with a C compiler (e.g., _`gcc -o pan pan.c`_).
- Run the verifier. One can run it using the shell command _`./pan`_. It reports whether all computations are correct or there exists a computation that contains an error. If an error is found, the verifier produces a trail file which can be used to reconstruct the erroneous computation. We'll make use of trail files shortly, to extract valid puzzle solutions.


We now turn to an example that introduces two Spin concepts, blocking expressions and nondeterminism, which we’ll use to solve the puzzle.

The first concept is a blocking expression that blocks execution when it evaluates to false. In Spin terms, this means the process that was running cannot move on to its next statement. For example, the following statement blocks execution when _`x`_ equals 2:

```
! (x == 2);
```

The statement above lets the process move forward only if the condition is true. The ! symbol means ‘negation,’ so the statement is true when _`x`_ is not equal to 2. So, the process continues only when _`x`_ is not 2.

The second concept is how nondeterminism and error search work in Spin. Consider the following small program (All Spin code in this post was written using Spin version 6.5.2):

<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/second.png" />

We wrap our code in a process-like executable unit using the _`proctype`_ keyword in Promela. When we prefix a _`proctype`_ definition with the _`active`_ keyword, we are telling the model checker to automatically start that process when Spin begins execution. 

The _`if`_ statement as written above (starting with the keyword _`if`_ and ending with _`fi`_) is how nondeterministic choice is expressed in Promela and Spin. Inside an _`if`_ statement, each option starts with `::`. The first statement after `::` is called a guard. If more than one guard statement is executable, Spin chooses one of them nondeterministically. In this case, the _`if`_ block assigns a value to _`x`_ nondeterministically. After this _`if`_ statement is executed, _`x`_ will have one of the values: 1, 2, 3, 4, or 5.

To further understand how this nondeterminism will play out, below is an automaton in _Figure 1_ that represents the possible transitions for this code (generated using a tool called jSpin [2]):

<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/three.png" />

_Figure 1_ 

The numbers inside the circles represent line numbers. As you can see, there is not just one computation path from start to finish, there are five possible computations, depending on which value is assigned to _`x`_.

At the end of the above program, we have an assert that checks whether `x == 1`. Will this assertion pass? It will fail, because Spin finds a counterexample. 

How does it find a counterexample? The assert causes the model checker to backtrack and explore all possible counterexamples from all computations. It tries to find computations where _`x`_ is not set to 1. For example, from five computations shown in the above figure, it will find paths where _`x`_ is set to 3, 4, or 5—but not 2, because that choice was blocked earlier using the guard expression.

Here’s a truncated version of the assertion violation report you would see when you run your model (assuming the above code is saved in a file named _atest.pml_):

```
$ spin -a atest.pml
$ gcc -o pan pan.c
$ ./pan -E         
pan:1: assertion violated (x==1) (at depth 1)
pan: wrote atest.pml.trail
```

The `-E` switch tells the verifier to ignore computations that end in invalid status which could be caused by blocked computations.
As indicated in the above output, this generates a trail file. We can replay it with Spin’s `-t -p` switches, inspect the output, and see how the computation led to the error. The output below, truncated for clarity, shows the sequence of steps that led to the assertion failure.

```
$ spin -t -p atest.pml

  1:    proc  0 (P:1) atest.pml:7 (state 3)     [x = 3]
  2:    proc  0 (P:1) atest.pml:12 (state 8)    [(!((x==2)))]
spin: atest.pml:13, Error: assertion violated
```

The output above shows that the assertion failed in the computation where _`x`_ was set to 3.  But there are other counterexamples that would also cause the assertion to fail, for example, when _`x`_ is set to 4 or 5. So how do we find all these counterexamples? We can do that using the following command:

```
./pan -E -c0 -e    
pan:1: assertion violated (x==1) (at depth 1)
pan: wrote atest.pml1.trail
pan: wrote atest.pml2.trail
pan: wrote atest.pml3.trail
```

For each counterexample, Spin generates a separate trail file (e.g., _atest.pml1.trail_, etc). We can replay these trail files to reconstruct the exact sequence of steps that led to the assertion failure. For example, let’s inspect the contents of _atest.pml3.trail_ to see another such erroneous computation:

```
$ spin -p -t -k atest.pml3.trail atest.pml

  1:    proc  0 (P:1) atest.pml:9 (state 5)     [x = 5]
  2:    proc  0 (P:1) atest.pml:12 (state 8)    [(!((x==2)))]
spin: atest.pml:13, Error: assertion violated
```

We see that when `x` was set to 5, it refuted our assertion claim. We can similarly replay the trail for other files with commands _`spin -p -t -k atest.pml1.trail atest.pml`_ and _`spin -p -t -k atest.pml2.trail atest.pml`_.

The key takeaway is this: we nondeterministically set _`x`_, wrote an assertion, and let the model checker backtrack and explore all possible computations involving the different values _`x`_ could take and the model checker found all computations that violated the assertion and reported them as counterexamples.

This ability of Spin to take code with nondeterminism and systematically backtrack and explore all possible computations in order to find a counterexample to an assertion is central to how we’ll find all solutions to the puzzle [3].

So how do we find those valid combinations that are solutions to LinkedIn Queens puzzle?

_Figure 1_ helped me come up with a solution. In that figure, we chose a value for _`x`_ nondeterministically from five options. The model checker then explored all five possible computations, one for each choice. Similarly, if we let the model checker nondeterministically pick one cell from each region, while enforcing the puzzle rules, and then backtrack through all possibilities, we can find all valid solutions.

To visualize this, I found it helpful to imagine a tree-like structure, as shown in _Figure 2_ below (If the image appears unclear, please consider opening it in a new tab.). It’s of course not ideal, but it helped me think through the problem. For both _Figure 2_ and _Figure 3_ below, we use the same board and colored region configuration as in the Python code above—where 1s represent region colored red, 2s represent region colored green, 3s represent region in yellow, and 4s represent region in pink. The board layout looks like this:
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

We could now imagine a tree with four levels. At level 0, you see all the cells from Region 1. From each of those cells, there are branches leading to every cell in Region 2.
In the figure, in Region 1, we show a branch going out from cell 1 just to save space—but you can imagine similar branches coming from cells 2, 3, and 4 as well. And the same for all cells in level 1 and 2. 

<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/four.png" />

_Figure 2_ 

Now let’s see what’s going on in Figure 2. A queen is first placed in cell 1, and then the search moves on to Region 2, where cells 5 and 6 are ignored because of the constraints. It places the queen in cell 7. The board at this point is shown on the right. When it moves to Region 3, it hits a dead-end—there is no cell available where the queen can be placed without violating the constraints. 

At that point, we could imagine Spin backtracking. Let’s look at the following illustration to see what happens next. 


<img loading="lazy" src="{{ site.baseurl }}/images/2025-08-18-linkedin-puzzle/five.png" />

_Figure 3_ 

After hitting a dead-end, it backtracks to Region 1 (visualized as the red arrow going from 7(Q) back to 2(Q)). It places the queen in cell 2 in region 1 (shown as 2(Q), colored in green). After successfully placing a queen there, it moves on to region 2, where it rejects cell 6 due to a column conflict, and cells 5 and 7 due to diagonal clashes. It then selects cell 8. In region 3, given all constraints, it places the queen in cell 9, and finally, in region 4, respecting constraints, it finds a valid position at cell 15, thus completing one valid solution in compliance with the row, column, and region constraints (the solved board is shown at the right).

Now let’s begin working toward a solution in Spin, with the goal of finding all possible solutions to the puzzle. We’ll solve it first for a `4x4` grid as it’s simple, and then the same code, with minor changes, would be used to solve it for a `9x9` grid. 

We use the same board and colored regions configuration as given above. We encode the region of 1s as:
```promela
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

```promela
inline ChooseRegion2() {
    if
    :: cell = 5    
    :: cell = 6   
    :: cell = 7    
    :: cell = 8   
    fi
}
```

And so on. Why make the model checker choose cells nondeterministically? We can connect this to _Figures 2 and 3_ and the explanation that followed. By organizing regions and their cells as shown in these figures you can see how this structure shows how the model checker might systematically explore all combinations.

We also define a few variables at the top of the model:

```
#define N 4
byte result[N];    
bool rows[N];      
bool cols[N];           
bool regions[N];
```

The _`result`_ array stores the final solution. Each _`result[region-1]`_ holds the cell where a queen is placed in that region. For example, `result[0] = 2` means the queen is placed in cell 2 in region 1.

The array _`rows`_ keeps track of whether a queen has already been placed in a given row. For example, _`rows[i]`_ is true if there is a queen in row _`i`_. Similarly, _`cols`_ keeps track of whether a queen has been placed in a given column. 

We process the board region by region, and record each chosen cell in the _`result`_ array. 

The general outline of what happens in the program is as follows. As the program processes the board region by region iteratively, from region 1 to 4, it performs these steps in sequence (we’ll go over each in some detail shortly):

- Iterate through all regions and do the following in each iteration:
    - Choose cells for a region nondeterministically.
    - Checks whether a queen is already placed in the current row and column. If we have, we block the process.
    - Ensures that we are not placing a queen on an adjacent diagonal. If we are, the process is blocked.
    - Mark the row and column as processed.
    - Places the queen and records its position in the _`result`_ array.
-  After exiting the loop, we use `assert(false)` at the end to make the model checker backtrack and explore all computations that successfully exit the loop. 

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

The program then checks and blocks if a queen has already been placed in the chosen row. This is done using a blocking expression:
```
!rows[row];  
```

As we discussed earlier, the execution continues if above is true, meaning the row is still available. It’s important to note that, because of these blocking statements, many computations will be blocked, but some won’t. To find valid solutions to the puzzle, we’ll make use of the assertion claim. More on this in a bit.

After checking the row, we similarly block the placement if a queen has already been placed in the same column. We also block the move if a queen has already been placed in this region. 

For the adjacency constraint, before placing a queen we check the immediate diagonal neighbors—upper-left, upper-right, lower-left, and lower-right. If any of those cells already contain a queen, the move is blocked. 

Once a valid cell is found, we mark the row, column, and region as occupied and we store the queen’s position in the _`result`_ array:

```
rows[row] = true;
cols[col] = true;      
result[region-1] = cell;
```

At this point, we’ve finished processing the region, so we move on to the next by restarting the loop. This process repeats until all regions have been processed.

As you can see, near the end, after the loop, we’ve added this line:
```
assert(false); 
```

This causes the model checker to backtrack and search for computations that successfully exit the loop, meaning computations that have placed queens on the board according to the rules. Any such computation is not just a counterexample to our assertion claim, but also a valid solution to the puzzle. [Here is the code](https://github.com/wyounas/linkedin_queens/blob/main/queenfourbyfour.pml) for `4x4` grid. 

Now to find a solution, we run the model and then run the verifier. 

You’ll see an assertion violation, along with a message indicating that a trail file has been written:
```
$ spin -a queenfourbyfour.pml
$ gcc -o pan pan.c
$ ./pan -E
pan:1: assertion violated 0 (at depth 79)
pan: wrote queenfourbyfour.pml.trail
```

Replay the trail file with command:
```
$ spin -p -t -k queenfourbyfour.pml.trail  queenfourbyfour.pml
```

The above will generate output that includes the contents of the _`result`_ array, our solution to the puzzle. Somewhere near the end of the generated output, you’ll see the _`result`_ array printed:

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

But this gives us just one solution to the puzzle. How can we find all solutions, ideally without adding any more code to our existing model? As we did earlier, by running _`./pan -E -c0 -e`_. You’ll get two solutions to the puzzle in respective trail files (i.e., _queenfourbyfour.pml1.trail_ and _queenfourbyfour.pml2.trail_). Here is the second solution:

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

Without changing much code, we can extend these ideas to a `9x9` grid. Here is a LinkedIn Queens puzzle by LinkedIn on a day in July 2025:

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

I took the original image and, based on the values in the _`result`_ array, placed queens (marked as "Q" above) in the image. It looks like a valid solution to the puzzle that satisfies all the constraints. (Please ignore the small ‘x’ in the image above, it was just a placeholder in the original image.)

Just for fun and curiosity, I wanted to find all solutions to the puzzle without the region constraint, that is, to find all valid queen placements where exactly one queen is placed in each row and column, and no two queens are adjacent, including diagonally.

[This code](https://github.com/wyounas/linkedin_queens/blob/main/queens_wo_region.pml) does that, and it reports 5242 solutions to the puzzle when we drop the region constraint for a `8 x 8` grid (please note that if you run this, it'll generate as many trail files as there are solutions).

In the end, I’d say that Spin’s support for nondeterminism is quite powerful and it can be leveraged to solve certain kinds of problems in an elegant way.

## Solving the puzzle with FizzBee


There’s another promising model checker on the horizon called [FizzBee](https://fizzbee.io/). Its syntax is close to Python, which makes it approachable for Python programmers.

To find a solution using FizzBee, I applied similar ideas to those I used with Promela and Spin.

Just as we wrap the code we want to explore in a _`proctype`_ in Promela to model process-like behavior, in FizzBee we use an _`action`_ block to define such executable units.

We begin with the _Init_ action, a special action that runs once at the start. It sets up the state. Here is what _Init_ looks like in our case:

```
action Init:
    N = 9  # 9x9 board
    # Define REGIONS as a list of cell IDs; omitted here for brevity
    #Also, we map each cell to (row, col) in the code
    board = [[0 for _ in range(N)] for _ in range(N)]

    row_queen_placed_in = [False for _ in range(N)]
    col_queen_placed_in = [False for _ in range(N)]
    region_queen_placed_in = [False for _ in range(N)]
    diagonals_blocked = [[False for _ in range(N)] for _ in range(N)]
    region_idx = 0
    done = False 
```

After that, the main search for solutions happens inside the _PlaceQueens_ action.

_PlaceQueens_ first checks whether all regions are filled; if so, it sets _done_ to true and returns. Otherwise, it processes the regions one by one: for each region it picks a candidate cell, checks the constraints, places a queen if allowed, updates the bookkeeping, and then advances to the next region. 

Let’s take a quick look at the FizzBee constructs we use as we process each region in turn.

We start a loop to iterate over each region. Then to pick a cell, we use the _any_ keyword. This makes the model checker explore every possible cell in the current region. This allows FizzBee to systematically explore valid combinations of cell placements across all regions.

```

atomic action PlaceQueens:
    if all([queen for queen in region_queen_placed_in]):
        done = True 
        return 
    
    while region_idx < N:
        cells_in_region = REGIONS[region_idx]
        cell = any cells_in_region

        row = (cell - 1) // N
        col = (cell - 1) % N
```


We then use _require_ guards to enforce all puzzle rules. Each guard ensures that the row, column, and region are free, and that no queen is placed on any of the four immediate diagonals. If any guard fails, the action stops; otherwise, we place the queen and mark the row, column, and region as used. We also then block the 
four immediate diagonals so they can't be used. 
```
    
    require row_queen_placed_in[row] == False  # row must be free
    require col_queen_placed_in[col] == False  # col must be free
    require region_queen_placed_in[region_idx] == False  # region must be free
    require diagonals_blocked[row][col] == False  # cell must not be diagonally blocked
    
    # commit placements
    board[row][col] = 1
    row_queen_placed_in[row] = True
    col_queen_placed_in[col] = True
    region_queen_placed_in[region_idx] = True
    
    # block the four immediate diagonal neighbors
    if row > 0 and col > 0:
        diagonals_blocked[row-1][col-1] = True # upper-left
    if row > 0 and col < N-1:
        diagonals_blocked[row-1][col+1] = True # upper-right
    if row < N-1 and col > 0:
        diagonals_blocked[row+1][col-1] = True # lower-left
    if row < N-1 and col < N-1:
        diagonals_blocked[row+1][col+1] = True # lower-right

```

FizzBee also lets us specify safety properties, properties that state a ‘bad thing never happens.’ We write these properties using the _always_ keyword. For example, to express that a variable _temperature_ should never be greater than zero, we can write:

```
always assertion AlwaysLessThanEqualsZero:
  return temperature <= 0
```

To check this invariant, the model checker looks for counterexamples where _temperature_ is greater than zero. If no such counterexample exists, the invariant holds and the model is considered correct. 

Here’s a small FizzBee example where we return false if the temperature is greater than zero. Since the temperature is set to 5 in the _Init_ action, the invariant fails, false is returned, and FizzBee reports this as an invariant violation, with a counterexample trace showing us the state where the invariant did not hold.

```
# A simple example showing how returning False triggers a counterexample

action Init:
    temperature = 5

always assertion AlwaysLessThanEqualsZero:
    if temperature <= 0:
        return True 
    return False 
```

When we run the above model, FizzBee reports:

```
FAILED: Model checker failed. Invariant:  AlwaysLessThanEqualsZero
------
Init
--
state: {"temperature":"5"}
------

```
In our puzzle, the same logic applies. We define the invariant as an “invalid board.” When that invariant fails, meaning none of the invalid conditions hold, FizzBee reports a counterexample, which is a valid solution to the puzzle.


An invalid board means these things:

1. Not all queens are placed,
2. A row with more than one queen,
3. A column with more than one queen,
4. A pair of queens touching diagonally, or
5. A region that doesn’t contain exactly one queen.

If any of these conditions hold, the board is invalid. A counterexample to this assertion is a valid solution. We check for these assertions in the invariant section that begins with the _always assertion_ keyword. When none of these invalid-board conditions hold, the invariant returns _False_. FizzBee then reports a counterexample, a board that violates the “invalid board” assertion, which represents a valid solution to the puzzle.

When I run this [code](https://github.com/wyounas/linkedin_queens/blob/main/queens.fizz) for the above `9x9` LinkedIn Queens puzzle, FizzBee returns one counterexample. You can find it by checking the value of the _board_ variable in the output. And the counterexample corresponds to a valid LinkedIn Queens solution.

Below is the resulting `9×9` board, taken from the _board_ variable in the counterexample. A value of 1 marks a cell where a queen is placed.

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

So far, I’ve only been able to get FizzBee to output a single counterexample as a solution even when I use a board that has multiple solutions. Currently, FizzBee doesn’t list all counterexamples. However, JP (the creator of FizzBee, who is always helpful) showed me an alternative approach. He suggested printing the board inside the assertion code at the end and returning true instead of false.

_Please keep in mind that I’m only human, and there’s a chance this post contains errors, even though I’ve done my best to avoid them. If you notice anything off, I’d appreciate a correction. Please feel free to [send me an email](mailto:waqas.younas@gmail.com)._


## References

1. PyEDA: [https://pyeda.readthedocs.io/en/latest/overview.html](https://pyeda.readthedocs.io/en/latest/overview.html )
2. jSpin: [https://github.com/motib/jspin](https://github.com/motib/jspin)
3. Ben Ari's book introduced me to nondeterminism in Spin. 
[Ben-Ari, M. (2008). Principles of the Spin model checker. Springer.](https://link.springer.com/book/10.1007/978-1-84628-770-1)
4. Repository containing all code in this post: [https://github.com/wyounas/linkedin_queens](https://github.com/wyounas/linkedin_queens)