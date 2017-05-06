## Don't forget to check for errors

```go
func WriteFile(filename string, data []byte) error {
  f, err := os.Create(filename)

  if err != nil {
    return err
  }

  defer f.Close()

  _, err = f.Write(data)
  return err
}
```

```go
func WriteFile(filename string, data []byte) (err error) {
  f, err := os.Create(filename)

  if err != nil {
    return err
  }

  defer func() {
    if cerr := f.Close(); cerr != nil && err == nil {
      err = cerr
    }
  }()

  _, err = f.Write(data)
  return err
}
```

```go
func safeClose(c io.Closer, err *error) {
  if cerr := c.Close(); cerr != nil && *err == nil {
    *err = cerr
  }
}

func WriteFile(filename string, data []byte) (err error) {
  f, err := os.Create(filename)

  if err != nil {
    return err
  }

  defer safeClose(f, &err)

  _, err = f.Write(data)
  return err
}
```

```go
// WriteFile writes data to a file named by filename.
// If the file does not exist, WriteFile creates it with permissions perm;
// otherwise WriteFile truncates it before writing.
func WriteFile(filename string, data []byte, perm os.FileMode) error {
  f, err := os.OpenFile(filename, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, perm)
  if err != nil {
    return err
  }
  n, err := f.Write(data)
  if err == nil && n < len(data) {
    err = io.ErrShortWrite
  }
  if err1 := f.Close(); err == nil {
    err = err1
  }
  return err
}
```
