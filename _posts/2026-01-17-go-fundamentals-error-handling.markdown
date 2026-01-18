---
date: 2026-01-17 12:00:04
title: "Go Fundamentals: Error Handling"
description: Understanding error values, error wrapping, and multiple errors in Go
image: images/posts/go-fundamentals/go-fundamentals-error-handling.png
categories: [go]
tags: [go, fundamentals, errors, error-handling]
series: go-fundamentals
---

This post covers Go error handling, including error values, error wrapping for context, and working with multiple errors.

<div class="series-nav">
  <div class="series-nav__title">Go Fundamentals Series</div>
  <div class="series-nav__subtitle">Article 4 of 4</div>
  <div class="series-nav__list">
    <a href="/go-fundamentals-variable-behavior" class="series-nav__item">1. Variable Behavior</a>
    <a href="/go-fundamentals-memory-architecture" class="series-nav__item">2. Memory Architecture</a>
    <a href="/go-fundamentals-concurrency-model" class="series-nav__item">3. Concurrency Model</a>
    <span class="series-nav__item--current">4. Error Handling ← You are here</span>
  </div>
</div>

## Error Values

In Go, error is just a regular variable. There is no mechanism that handles the control flow of errors (like try-catch, for example). Deciding how to act upon errors is explicitly handled in every function.

This will probably look familiar:

```go
err := someFunc()
if err != nil {
    return err
}
// ...
```

In some cases certain errors need specific handling decisions:

```go
func getUserData(userID string) (responseStatus int, response string) {
    if userID == "" {
        return http.StatusBadRequest, "invalid user id"
    }

    value, err := redisDB.Get(context.Background(), userID).Result()
    if err != nil {
        if err == redis.Nil {
            return http.StatusOK, "user has no data"
        } else {
            return http.StatusInternalServerError, "internal server error"
        }
    }

    return http.StatusOK, value
}
```

There are 3 types of error values:
- Sentinel errors
- Error types
- Anonymous errors

### Sentinel Errors

Sentinel errors are just global variables indicating an error.

- [+] They provide the best performance
- [+] They provide the easiest error handling decisions
- [-] They don't allow specifying incident specific values

```go
var ErrInvalidUser = errors.New("invalid user")

func producerExample(userID string) error {
    user, err := getUser(userID)
    if err != nil {
        return err
    }

    if user == nil {
        return ErrInvalidUser
    }

    return nil
}

func consumerExample() {
    err := producerExample("user123")
    if err != nil {
        if err == ErrInvalidUser {
            log.Info("user doesn't exist")
        } else {
            log.Errorf("got error %v", err)
        }
    }
}
```

### Error Types

Error types are types implementing the error interface.

```go
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
    Error() string
}
```

- [+] They allow specifying incident specific values
- [-] They have poor performance
- [-] They provide error handling decisions, but they require a bit more code

```go
type InvalidUserError struct {
    UserID string
}

func (i *InvalidUserError) Error() string {
    return fmt.Sprintf("user %s is invalid", i.UserID)
}

func producerExample(userID string) error {
    user, err := getUser(userID)
    if err != nil {
        return err
    }

    if user == nil {
        return &InvalidUserError{UserID: userID}
    }

    return nil
}

func consumerExample() {
    err := producerExample("user123")
    if err != nil {
        if e, ok := err.(*InvalidUserError); ok {
            log.Infof("user %s doesn't exist, message: %v", e.UserID, e.Error())
        } else {
            log.Errorf("got error %v", err)
        }
    }
}
```

### Anonymous Errors

Anonymous errors are error values returned but aren't directly identifiable.

- [+] They allow specifying incident specific values
- [-] They have poor performance
- [-] No effective error handling decisions

This is less code to write, but it has bad performance and does not allow handling decisions.

```go
func producerExample(userID string) error {
    user, err := getUser(userID)
    if err != nil {
        return err
    }

    if user == nil {
        return fmt.Errorf("user %s is invalid", userID)
    }

    return nil
}

func consumerExample() {
    err := producerExample("user123")
    if err != nil {
        if strings.Contains(err.Error(), "is invalid") {
            log.Info("user doesn't exist, message: %v", err.Error())
        } else {
            log.Errorf("got error %v", err)
        }
    }
}
```

### Error Values Summary

Error values can be sentinel, custom types, and anonymous.

Use sentinel errors when possible. Otherwise prefer custom types due to better error handling decisions. Use anonymous errors only if internal code and no error handling decisions are needed.

## Error Wrapping

### Motivation

Consider the following code:

```go
func (u *UserService) AddUser(email, password, firstName, lastName string) {
    err := u.saveUser(email, password, firstName, lastName)
    if err != nil {
        logrus.Errorf("could not save user: %s", err.Error())
        return
    }

    err = u.sendWelcomeEmail(email, firstName)
    if err != nil {
        logrus.Errorf("could not send welcome email: %s", err.Error())
        return
    }
}

func (u *UserService) saveUser(email, password, firstName, lastName string) error {
    valid, err := u.isValidEmail(email)
    if err != nil {
        return err
    }

    if !valid {
        return ErrUserAlreadyExists
    }

    return u.storeUser(email, password, firstName, lastName)
}

func (u *UserService) isValidEmail(email string) (bool, error) {
    exists, err := u.emailDB.EmailExists(email)
    if err != nil {
        return false, err
    }

    if exists {
        return false, nil
    }

    err = u.emailDB.StoreEmail(email)
    if err != nil {
        return false, err
    }

    return true, nil
}

func (u *UserService) storeUser(email, password, firstName, lastName string) error {
    err := u.userDB.InsertUserPassword(email, password)
    if err != nil {
        return err
    }

    return u.userDB.InsertUserDetails(email, firstName, lastName)
}

func (u *UserService) sendWelcomeEmail(email, firstName string) error {
    return u.emailService.Send(email, "welcome, " + firstName)
}
```

Two interesting things happen in Go's error handling:

1. Due to explicit control flow management, errors must be propagated to the top-most level affected by them.
2. Due to lack of stack traces, error messages do not contain context.

![Error Propagation](/images/posts/go-fundamentals/error-propagation.png)

If the error is simply `redis: nil`... where? On which query? On which function? Where did it start?

### Attempt #1: Replace all errors with custom errors

```go
func (u *UserService) saveUser(email, password, firstName, lastName string) error {
    valid, err := u.isValidEmail(email)
    if err != nil {
        return ErrCouldNotValidateEmail
    }

    if !valid {
        return ErrUserAlreadyExists
    }

    err = u.storeUser(email, password, firstName, lastName)
    if err != nil {
        return ErrCouldNotStoreEmail
    }

    return nil
}
```

This explains where the error is coming from. But now we lost the actual error. Was it a networking issue? Was it a `redis: nil` response?

### Attempt #2: Replace all errors with custom errors containing the original error

```go
type SaveUserError struct {
    task string
    err  error
}

func (s SaveUserError) Error() string {
    return fmt.Sprintf("could not save user, failed to %s due to %s", s.task, s.err.Error())
}

func (u *UserService) saveUser(email, password, firstName, lastName string) error {
    valid, err := u.isValidEmail(email)
    if err != nil {
        return SaveUserError{task: "validate email", err: err}
    }

    if !valid {
        return ErrUserAlreadyExists
    }

    err = u.storeUser(email, password, firstName, lastName)
    if err != nil {
        return SaveUserError{task: "store user", err: err}
    }

    return nil
}
```

This is better. Now the error message is clear:

```
ERRO[0000] could not save user: could not save user, failed to validate email due to redis: nil
```

Also, we can perform error handling decisions on `SaveUserError`. But can we perform error handling decisions on `redis.Nil`?

```go
func (u *UserService) AddUser(email, password, firstName, lastName string) {
    err := u.saveUser(email, password, firstName, lastName)
    if err != nil {
        if err == redis.Nil {
            // TODO alert our dedicated redis experts team who will happily assist
        }
        if e, ok := err.(*SaveUserError); ok {
            // TODO alert our enthusiastic user-management team who are always on the lookout for such issues
        }
        logrus.Errorf("could not save user: %s", err.Error())
        return
    }
}
```

- Checking `err == redis.Nil` - This won't work because `err` is a `SaveUserError`, not `redis.Nil`
- Checking `err.(*SaveUserError)` - This will work

### Attempt #3: Use error wrapping!

Add an `Unwrap` method to expose underlying errors.

```go
type SaveUserError struct {
    task string
    err  error
}

func (s SaveUserError) Error() string {
    return fmt.Sprintf("could not save user, failed to %s due to %s", s.task, s.err.Error())
}

func (s SaveUserError) Unwrap() error {
    return s.err
}
```

Now both checks work:

```go
func (u *UserService) AddUser(email, password, firstName, lastName string) {
    err := u.saveUser(email, password, firstName, lastName)
    if err != nil {
        if errors.Is(err, redis.Nil) {
            // TODO alert our dedicated redis experts team who will happily assist
        }
        var saveErr *SaveUserError
        if errors.As(err, &saveErr) {
            // TODO alert our enthusiastic user-management team who are always on the lookout for such issues
        }
        logrus.Errorf("could not save user: %s", err.Error())
        return
    }
}
```

If you choose anonymous errors, you can use the `%w` verb with `fmt.Errorf`:

```go
func (u *UserService) saveUser(email, password, firstName, lastName string) error {
    valid, err := u.isValidEmail(email)
    if err != nil {
        return fmt.Errorf("could not validate email due to %w", err)
    }

    if !valid {
        return ErrUserAlreadyExists
    }

    err = u.storeUser(email, password, firstName, lastName)
    if err != nil {
        return fmt.Errorf("could not store user due to %w", err)
    }

    return nil
}
```

### Error Wrapping Summary

Wrap errors to add more context to error messages. This might be needed when errors are propagated upwards.

Wrap by implementing the `Unwrap` method, or by using the `fmt.Errorf` `%w` verb.

Perform error handling decisions using `errors.Is` and `errors.As` to ensure recursive unwrapping.

## Multiple Errors

Starting at Go 1.20 you can return multiple errors using `errors.Join`:

```go
func GetAllURLs(urls []string) ([]*http.Response, error) {
    var errs error
    result := make([]*http.Response, 0, len(urls))
    for _, url := range urls {
        response, err := http.Get(url)
        errs = errors.Join(errs, err)
        if err == nil {
            result = append(result, response)
        }
    }
    return result, errs
}
```

`errors.Join` automatically validates all errors and eventually returns `nil` if no errors occurred. It provides an elaborate error message in case any errors occurred. And finally, it provides unwrapping capabilities to identify wrapped errors and provide error handling decisions.
