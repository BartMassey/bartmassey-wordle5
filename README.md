# wordle5: Fast Solver For Wordle5 Puzzle
Bart Massey 2022

## Background

YouTuber Matt Parker proposed an interesting problem
inspired by the game *Wordle* a while back: find a set of
five five-letter words from a dictionary that collectively
contain 25 of the 26 letters of the English alphabet.

For a summary history of the problem, see the `README` in
Philip Meier's
[solution repo](https://github.com/pmeier/parker-word-puzzle):
this solution inspired me to create my own solution in Rust;
specifically, I started by responding to this
[Reddit thread](https://www.reddit.com/r/learnrust/comments/x5ykmt/comment/in7l45g/).

This program currently solves the problem in about 8ms for
me. See the [Performance](#performance) section below for
much more information.

## Algorithm

My approach is "brute-force" state-space search, as is the
standard method for solvers for this. (Apparently somebody
used a graph clique finder, which is *tres* cool, but I
can't imagine it's faster.)

The key is in selection of state space and in pruning. For
the state space I choose to search for words by containing
letter from "hardest" letter to "easiest": a letter is
considered harder if it has fewer words containing it. I
thus start my depth-first search by selecting a word
containing a `q` and proceeding from there.

We may and must use only 25 of the 26 letters in our
solution. This complicates the search space a bit: we must
allow for "skipping" a letter during the search.

Several kinds of pruning are applied this search.

* *Reuse Checking:* This is the simplest case: a word is not
  considered if previously-selected words contain any of its
  letters. Any search that fails to do this is ridiculous.
  Reuse checks are what makes searching from hardest to
  easiest letter a good candidate for the search space.

* *Candidate Filtering:* The word list for each letter is
  filtered before starting the search to remove words that
  can't possibly work given the previous known-chosen
  letters. If a word *w* at a given candidate letter *l* in
  the search contains more than one letter that was
  previously selected, *w* is removed from the word list for
  *l*. (The search order is statically known, making this
  calculation possible.)

  Words with one previously selected letter may still be
  candidates: because there are only 25 letters to be
  selected, that letter may have been skipped in the search.
  (I experimented with building separate zero-overlap and
  one-overlap candidate word lists, but it did not show a
  speed improvement.)

* *Pseudovowel Pruning:* "Pseudovowels" for a word list
  *l* are sets *p* of letters such that every word in *l*
  must contain at least one letter in *p*. For the
  dictionaries used here, the set of vowels "aeiouwy"
  obeys this property. The algorithm automatically
  calculates a set of pseudovowels for the given dictionary
  up front: the current approach is to just take the letters
  in decreasing dictionary frequency until they are seen to
  be pseudovowels by examining the dictionary.  For the NYT
  Wordle dictionary, the program uses pseudovowels
  "aeilnorsu".

  Having a set of *n* pseudovowels for a dictionary allows
  filtering out words that contain more than
  *n*&nbsp;-&nbsp;5 pseudovowels up front.

  Given a set of *n* pseudovowels, the program can check
  during the search that these pseudovowels have not been
  "overused". If there are fewer than *n - (5 - d)*
  pseudovowels remaining after selecting a word *w* at depth
  *d*, there are not enough remaining pseudovowels to make
  the remaining words. Thus, *w* can be omitted from the
  search.

  This version of the program finds the standard vowels
  "aeiouyw" itself and uses them as pseudovowels. This is
  "fair" because the standard vowels are being discovered by
  the program rather than using expert knowledge.

The resulting algorithm looks something like this:

<!-- This pseudocode translated from algorithm.pseu by pseuf -->

> Calculate&nbsp;and&nbsp;sort&nbsp;a&nbsp;sequence&nbsp;of&nbsp;letter groups&nbsp;*g*&nbsp;and&nbsp;a&nbsp;set&nbsp;of&nbsp;pseudovowels&nbsp;*p*  
> To&nbsp;search&nbsp;at&nbsp;position&nbsp;*i*&nbsp;in&nbsp;*g*&nbsp;at&nbsp;depth&nbsp;*d*,  
> &nbsp;&nbsp;with&nbsp;a&nbsp;set&nbsp;of&nbsp;*seen*&nbsp;letters&nbsp;and&nbsp;a&nbsp;partial&nbsp;*soln*&nbsp;and&nbsp;a&nbsp;*skipped*&nbsp;indicator:  
> &nbsp;&nbsp;&nbsp;&nbsp;**if**&nbsp;*d*&nbsp;==&nbsp;5  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;save&nbsp;*soln*  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**return**  
> &nbsp;&nbsp;&nbsp;&nbsp;find&nbsp;the&nbsp;next&nbsp;position&nbsp;j&nbsp;after&nbsp;*i*&nbsp;with&nbsp;a&nbsp;not-seen&nbsp;*g*.*letter*  
> &nbsp;&nbsp;&nbsp;&nbsp;**for**&nbsp;each&nbsp;word&nbsp;*w*&nbsp;**in**&nbsp;*g*.*words*  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**if**&nbsp;*w*&nbsp;contains&nbsp;*seen*&nbsp;letters  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;continue  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**if**&nbsp;pseudovowel&nbsp;pruning&nbsp;eliminates&nbsp;*w*  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;continue  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;search&nbsp;at&nbsp;position&nbsp;*i*&nbsp;+&nbsp;1,&nbsp;depth&nbsp;*d*&nbsp;+&nbsp;1,&nbsp;*skipped*,  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;with&nbsp;updated&nbsp;*seen*&nbsp;and&nbsp;*soln*  
> &nbsp;&nbsp;&nbsp;&nbsp;**if**&nbsp;not&nbsp;*skipped*  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;search&nbsp;at&nbsp;position&nbsp;*i*&nbsp;+&nbsp;1,&nbsp;depth&nbsp;*d*,&nbsp;*skipped*&nbsp;true,  
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;with&nbsp;*seen*&nbsp;and&nbsp;*soln*&nbsp;(skip&nbsp;*g*.*letter*)  

<!-- End of pseuf translation of algorithm.pseu -->

## Implementation

The "obvious" performance-improving techniques have been
used in the implementation of this algorithm. Letters are
represented as integers 0..26. Letter sets are represented
as bitsets in 32 bits. Words are represented as letter sets
(since they are guaranteed to contain five unique letters).
The dictionary of text words is replaced with a dictionary
of letter-set words early on, with a map back to text kept
around to print the solution.

The "POPCOUNT" primitive, which counts the number of set
bits in a 32-bit word, plays a key role in performance
here. See my
[popcount micro-benchmarks](http://github.com/BartMassey/popcount)
repo for for an account of how much slower this operation
can be if not provided by your processor. Thank goodness
modern processors have finally incorporated the damn thing,
and that Rust's `count_ones()` intrinsic provides easy
access to it.

A bunch of stuff that might either slow things down or just
make them less convenient has been hidden behind Rust
`feature` gates and must be turned on at compile time. See
below for specifics.

## Performance

My solution is blazingly fast, solving the standard problem
in about 8ms single-threaded on my Ryzen 9 3900X desktop.

The comment thread on this
[YouTube video](https://youtu.be/Y37WiO55bxs) seems to be
the source of fastest solutions right now: I'm about 2.5×
faster than the next-best reported solution.

Timing shows that much of the runtime of the single-threaded
version is spent in the solver proper: about 2ms for init,
4ms for the solver, 2ms of unknown overhead. This leaves
little room for improvement by solver speedup.

The `main` branch code uses `std::fs::read_to_string()`
followed by line splitting of the string to read the
dictionaries. The branch `bufread` in this repo moves to
using `std::fs::open()` and `std::io::BufRead::lines()`. It
is dramatically slower, taking about 20ms extra just to
process the dictionaries. Rust I/O performance is a bit
wack.

The `main` branch uses a recursive solver. The branch
`nonrecursive-solvify` in this repo makes the solver
iterative using an explicit stack. It is not noticeably
faster currently, but that appears to be a function of the
already-rapid solver time.

The branch `no_std` in this repo allows building a `no_std`
version of the program. It is not faster than the `std`
version, but was sure more work to produce.

The branch `parallelized` in this repo includes both a
`rayon`-parallelized and custom scoped-thread-parallelized
version of the program. These provided less than a
millisecond of speedup, and thus were removed to simplify
the program.

When building for best performance, you may want to build a
statically-linked binary for more reproducible best times.
On my box I use the `x86_64-unknown-linux-musl` build target
for this. You may also want to use `RUSTC_FLAGS="-C
target-cpu=native"`, although it doesn't make a difference
for me. Note that you definitely want to time the binary:
don't use `cargo run` when timing as it adds major overhead.

To see node counts from the solver, build with the
`instrument` feature. This will display node counts at each
search tree depth as well as a total.

To get times for initialization and solver, build with the
`timing` feature. This will display the wall-clock time for
each of these pieces

At this point, the performance is really fragile; small
tweaks make hard-to-understand differences. Further, setup
time and search time are looking pretty balanced. I think
it's unlikely that further tuning of the existing approach
can make this code dramatically quicker: a whole new solver
algorithm would be needed.

In terms of overheads, the remaining possibilites are ugly.

* One could cheat massively by
  compiling the pre-digested dictionaries into the program to
  save a millisecond or two, but ugh.

* I tried going to `no_std` to get rid of the 2-3ms of
  startup overhead, thinking it was due to Rust startup This
  was a massive uglification of the code, and produced no
  noticeable speedup.

* I used parallelism for the longest time. As the program
  got faster, the speedups got microscopic. At this point we
  are probably limited by data access rather than CPU.

I've tried to make my solution clear and readable. Please
see the Rustdoc and source code for details.

## Word Lists

* `words-nyt-wordle.txt` (12945 words): I've taken the
  liberty of combining the NYT Wordle word lists into a
  single file and filtering duplicates. It doesn't change
  performance notably, and it's way more convenient.

* `words-alpha.txt` (15913 words): I've filtered
  `words_alpha.txt` (wherever that came from) to just
  five-letter words and removed all consonant-only words
  (all words contain at least one of *aeiouyw*).

* `words-bart.txt` (17560 words): I've built my own wordlist
  as the union of wordlists in
  [my `wordlists` repo](https://github.com/BartMassey/wordlists),
  filtered the same as `words-alpha.txt`.

## Usage

Build the program with
```
cargo build --release
```

Invoke the program with a list of the dictionary files to be
read. Dictionary files should consist of ASCII lowercase
words, one per line. The easy invocation is
```
cargo run --release words-nyt-wordle.txt
```

To get a pseudovowel list and node count instrumentation,
compile with feature `instrument`, for example
```
cargo run --release --features=instrument -- --sequential words-nyt-wordle.txt
```

To get timings for initialization and solve, compile with feature
`timing`, for example
```
cargo run --release --features=timing -- --sequential words-nyt-wordle.txt
```

## License

This work is made available under the "MIT License."  Please
see the file `LICENSE.txt` in this distribution for license
terms.  The provided dictionaries are used without
permission: no license is provided, express or implied, for
these.
