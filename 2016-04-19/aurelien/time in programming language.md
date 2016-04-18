https://www.youtube.com/watch?v=b0EF0VTs9Dc

35:00 Exceptions vs failure functions
- exceptions goes back in time: it searches for a previous good state of the program and restores it
- promises go forward in time: their accept a failure function that will be called later if a failure happen
36:00:
- exceptions modify the flow of control by unwinding the state
- in a turn based system, the stack is empty at the end of every turn
- exceptions cannot be delivered accross turns
