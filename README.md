# Demo of Concolic Fuzzing

Consider the following function
```python
def factorial(n):
    if n < 0:
        return None

    if n == 0:
        return 1

    if n == 1:
        return 1

    v = 1
    while n != 0:
        v = v * n
        n = n - 1

    return v
```
Is this sufficient to explore all the features of the function? How do we know? One way to verify that we have explored all features is to look at the** _coverage obtained_**.

## Tracking Constrains

we first define the `Coverage` class
```python
from types import FrameType, TracebackType
from typing import Any, Optional, Callable, Type, List, Tuple, Set

Location = Tuple[str, int]
class Coverage:
    """Track coverage within a `with` block. Use as
    ```
    with Coverage() as cov:
        function_to_be_traced()
    c = cov.coverage()
    ```
    """

    def __init__(self) -> None:
        """Constructor"""
        self._trace: List[Location] = []

    # Trace function
    def traceit(self, frame: FrameType, event: str, arg: Any) -> Optional[Callable]:
        """Tracing function. To be overloaded in subclasses."""
        if self.original_trace_function is not None:
            self.original_trace_function(frame, event, arg)

        if event == "line":
            function_name = frame.f_code.co_name
            lineno = frame.f_lineno
            if function_name != '__exit__':  # avoid tracing ourselves:
                self._trace.append((function_name, lineno))

        return self.traceit

    def __enter__(self) -> Any:
        """Start of `with` block. Turn on tracing."""
        self.original_trace_function = sys.gettrace()
        sys.settrace(self.traceit)
        return self

    def __exit__(self, exc_type: Type, exc_value: BaseException, 
                 tb: TracebackType) -> Optional[bool]:
        """End of `with` block. Turn off tracing."""
        sys.settrace(self.original_trace_function)
        return None  # default: pass all exceptions

    def trace(self) -> List[Location]:
        """The list of executed lines, as (function_name, line_number) pairs"""
        return self._trace

    def coverage(self) -> Set[Location]:
        """The set of executed lines, as (function_name, line_number) pairs"""
        return set(self.trace())

    def function_names(self) -> Set[str]:
        """The set of function names seen"""
        return set(function_name for (function_name, line_number) in self.coverage())

    def __repr__(self) -> str:
        """Return a string representation of this object.
           Show covered (and uncovered) program code"""
        t = ""
        for function_name in self.function_names():
            # Similar code as in the example above
            try:
                fun = eval(function_name)
            except Exception as exc:
                t += f"Skipping {function_name}: {exc}"
                continue

            source_lines, start_line_number = inspect.getsourcelines(fun)
            for lineno in range(start_line_number, start_line_number + len(source_lines)):
                if (function_name, lineno) in self.trace():
                    t += "# "
                else:
                    t += "  "
                t += "%2d  " % lineno
                t += source_lines[lineno - start_line_number]

        return t
```

And extend to provide us with coverage arcs
```python
import inspect
import sys

class ArcCoverage(Coverage):
    def traceit(self, frame, event, args):
        if event != 'return':
            f = inspect.getframeinfo(frame)
            self._trace.append((f.function, f.lineno))
        return self.traceit

    def arcs(self):
        t = [i for f, i in self._trace]
        return list(zip(t, t[1:]))
```
Next, we use the Tracer to obtain the coverage arcs.

```python
with ArcCoverage() as cov:
    factorial(5)
```

## Concolic Execution
One way to cover additional branches is to look at the execution path being taken, and collect the conditional constraints that the path encounters. Then we can try to produce inputs that lead us to taking the non-traversed path.

First, let us step through the function.
```python
lines = [i[1] for i in cov._trace if i[0] == 'factorial']
src = {i + 1: s for i, s in enumerate(
    inspect.getsource(factorial).split('\n'))}
```
The line (1) is simply the entry point of the function. We know that the input is n, which is an integer.

```python
src[1]
'def factorial(n):'
```
The line (2) is a predicate n < 0. Since the next line taken is line (5), we know that at this point in the execution path, the predicate was false.

```python
src[2], src[3], src[4], src[5]
('    if n < 0:', '        return None', '', '    if n == 0:')
```
We notice that this is one of the predicates where the true branch was not taken. How do we generate a value that takes the true branch here? One way is to use symbolic variables to represent the input, encode the constraint, and use an SMT Solver to solve the negation of the constraint.

## Solving Constrains
To solve these constraints, one can use a Satisfiability Modulo Theories (SMT) solver. An SMT solver is built on top of a SATISFIABILITY (SAT) solver. A SAT solver is being used to check whether boolean formulas in first order logic (e.g. (a | b ) & (~a | ~b)) can be satisfied using any assignments for the variables (e.g a = true, b = false). An SMT solver extends these SAT solvers to specific background theories.

We use the SMT solver [Z3](https://github.com/Z3Prover/z3#readme) in this chapter.

```python
import z3
```












