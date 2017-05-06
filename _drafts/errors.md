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

## Don't clutter your code with err != nil checks

```go
type gpsPoint struct {
	Longitude     float32
	Latitude      float32
	Distance      int32
	ElevationGain int16
	ElevationLoss int16
}
```

```go
func parseDataPoint(input io.Reader) (*gpsPoint, error) {
  var point gpsPoint

  if err := binary.Read(input, binary.BigEndian, &point.Longitude); err != nil {
    return nil, err
  }

  if err := binary.Read(input, binary.BigEndian, &point.Latitude); err != nil {
    return nil, err
  }

  if err := binary.Read(input, binary.BigEndian, &point.Distance); err != nil {
    return nil, err
  }

  if err := binary.Read(input, binary.BigEndian, &point.ElevationGain); err != nil {
    return nil, err
  }

  if err := binary.Read(input, binary.BigEndian, &point.ElevationLoss); err != nil {
    return nil, err
  }

  return &point, nil
}
```

```go
type reader struct {
  r   io.Reader
  err error
}

func (r *reader) read(data interface{}) {
  if r.err == nil {
    r.err = binary.Read(r.r, binary.BigEndian, data)
  }
}
```

```go
func parseDataPoint(input io.Reader) (*gpsPoint, error) {
  var point gpsPoint
  r := reader{r: input}

  r.read(&point.Longitude)
  r.read(&point.Latitude)
  r.read(&point.Distance)
  r.read(&point.ElevationGain)
  r.read(&point.ElevationLoss)

  if r.err != nil {
    return nil, r.err
  }

  return &point, nil
}
```
