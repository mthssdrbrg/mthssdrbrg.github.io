---
title: helpers in the kitchen
tags: test-kitchen chef rspec
---

When writing tests using RSpec, I prefer to use `shared_context` and
`shared_examples` for integration setup and for testing shared behaviour,
respectively.

When I started out with writing integration tests for my `kafka` cookbook using
`serverspec`, I wanted to share tests between different suites as they were
testing different init systems, but the tests could be heavily refactored to
just depend on some shared variables rather than duplicating all of the test
cases.
It was however not immediately clear how one would go about sharing files
between different suites.

After quite some digging I found a bit of information (I think in an old issue or
pull request though I no longer have any links handy) that mentioned having a
`helpers` directory in `test/integration` for sharing files between suites.

So it's just a matter of creating a `helpers` directory and the necessary
`busser` specific subdirectory, adding some files and you're good to go.
They'll even be available on the `$LOAD_PATH`, so it's easy to just require a
`spec_helper` or the alike in the actual `spec` files.

Since v1.2.1 of `test-kitchen` it's also possible to create directories in
the `helpers` directory (I tend to keep shared code in a `support` directory for
example).

For reference my `kafka` cookbook is [over
here](https://github.com/mthssdrbrg/kafka-cookbook), and more specifically the
`helpers` directory (and `serverspec` subdirectory) is [over here]
(https://github.com/mthssdrbrg/kafka-cookbook/tree/master/test/integration/helpers/serverspec).
