Removed the special `ErrNoRows` branch from both ensure methods. A missing
row now returns the same wrapped query error as any other read failure,
including the underlying error; it does not return a key.
