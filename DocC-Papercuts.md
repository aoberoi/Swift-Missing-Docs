# DocC Papercuts

## References

### Visibility of async functions and methods

When visiting any reference page that lists functions or methods, there is no indication whether a
particular function or method is an `async` function. You must click into the actual function or
method to see the full definition, and notice the `async` keyword just before the return type. This
is not prominent enough.

Example: https://developer.apple.com/documentation/foundation/urlsession#3842971. Methods such as
`data(for:delegate:)` are `async`, while `dataTask(with:)` are normal functions. Notice that the
normal function doesn't take a completion handler, which an experienced developer may misinterpret
for a synchronous function when scanning the listing, since there is a version that also takes a
completion handler.
