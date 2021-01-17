## novi-seccomp ##

A container-focused seccomp-cBPF generation library, intended primarily for use
within [runc][runc]. Currently [libseccomp][libseccomp] is missing several
features we need in order to have the behaviour we would like in container
runtimes, so the goal is to have our own cBPF generation code (while still
using libseccomp for all of its handling of syscalls and architecture-specific
semantics).

The primary issue being solved by this library is that libseccomp doesn't allow
filters to disambiguate "unknown" syscalls and "known" syscalls when
configuring a syscall allow-list. [This is a major issue for runc.][runc-2151].

libseccomp only has the concept of a single default action, which conflates
syscalls that have been intentionally blocked (usually handled with
`SECCOMP_RET_ERRNO(EPERM)`) with syscalls that were not included because the
profile author was not aware of them (which *should* be handled by
`SECCOMP_RET_ERRNO(ENOSYS)` in order to avoid causing problems for libcs on
Linux).

Our goal is to have any syscall which is newer than the latest syscall
specified in the profile (or alternatively, newer than a "minimum kernel
version" specified in the profile -- though this configuration knob doesn't
exist yet) will get `-ENOSYS` and all other syscalls will get `-EPERM` if there
is not an explicit rule which permits a different behaviour. Note that
`SECCOMP_RET_ERRNO(-EPERM)` would be user-configurable in this case, while the
`-ENOSYS` behaviour would not -- at least, not trivially.

Currently the minimum kernel version has been chosen to be Linux 3.0 (which
should be old enough that no userspace should be confused by `-EPERM` from a
syscall which has existed for that long). In future this will be configurable
(or auto-detected), but first libseccomp needs to [gain support for syscall
version information][libseccomp-11].

[runc]: https://github.com/opencontainers/runc
[libseccomp]: https://github.com/seccomp/libseccomp
[runc-2151]: https://github.com/opencontainers/runc/issues/2151
[libseccomp-11]: https://github.com/seccomp/libseccomp/issues/11

### Why not do [XYZ]? ###

Before working on this project, we attempted several workarounds in order to
solve this problem without writing our own cBPF generator. While there are
several workarounds which seem promising at first glance, they are not workable
(or they are deficient in some manner which may cause regressions for real
workloads).

The following is the most promising series of workarounds attempted:

 1. Setting the default errno to `-ENOSYS` rather than the historical `-EPERM`
    if the user requests `SECCOMP_RET_ERRNO` as the default action. This is the
    most trivial workaround, and its problems justify most of the other
    workarounds attempted.

    The primary issue is that this results in strange error codes being
    returned to userspace when they attempt to use a blocked syscall -- many
    userspace programs (including runc) have fallback code that detected
    `-EPERM` and `-EACCES` but few handle `-ENOSYS` in the same way in these
    cases.

    In addition, glibc masks `-ENOSYS` fairly regularly, returning other error
    codes like `-EINVAL` depending on the attempted syscall. This further
    complicates userspace fallback code, and will possibly cause even more
    confusion.

    The correct mental model to apply to seccomp filters is that they are like
    another LSM, and thus any denial from seccomp should be treated like an
    AppArmor or SELinux denial -- a return value of `-EPERM` (or `-EACCES`).
    Therefore `-ENOSYS` is not acceptable as a denial errno, because it means
    that userspace may attempt an incorrect fallback.

 2. In addition to the above, adding explicit `-EPERM` rules for any syscall
    not mentioned in the profile but did exist in the "minimum kernel version"
    of the profile.

    This does work, but it misses syscalls that have non-blanket allow rules
    (such as `clone(2)` in the default Docker profile -- where
    `clone(CLONE_NEWUSER)` and some other flags are blocked, but other
    `clone(2)` flags are not). These syscalls will get `-ENOSYS` which is
    really non-ideal and may result in strange userspace behaviour but only
    when certain syscall flags are used.

    Unfortunately libseccomp doesn't have the ability to specify a per-syscall
    fallback if not other rule matches the syscall, which would be the most
    trivial way of solving this problem. So more work is needed to avoid these
    problems.

 3. In addition to the above, generate "inverse rules" for any syscall which
    does appear in the filter but doesn't have blanket-allow rules.

    This covers more cases, but has several very large downsides:

     * Not all rules have legal inverse forms. libseccomp doesn't allow
       multiple comparisons to be applied to a single syscall argument within a
       rule, meaning that you cannot invert logical OR rules (by De-Morgan's
       laws the inverse becomes a logical AND).

       In addition, [there is no `MASKED_NEQ` operator][libseccomp-310] --
       though you can work around this by generating many `MASKED_EQ`
       comparisons (though the number of operations grows with the number of
       bits set in the mask).

       Therefore certain rules simply cannot be inverted and will have the
       `-ENOSYS` fallback -- which we want to avoid.

     * Syscalls can be specified more than once and the actions can be
       different for each rule -- meaning that very complicated logic may be
       necessary to figure out what is the precise set of inverse rules that
       need to be applied in order for the configured rules to work properly.

       The trivial response to this is to say "oh well, those filters are out
       of scope" but the default Podman filter has a similarly complicated
       filter for `socket(2)`.

An alternative workaround that was also attempted is simply to patch the cBPF
generated by libseccomp. This has several issues:

 1. It increases the potential for (severe security) bugs because now the
    filter generation code is non-obvious, and libseccomp doesn't have
    guarantees that their cBPF generation code will produce stable results
    between releases.

 2. The generated code is quite complicated and it's not really possible to
    distinguish user-specified rules and the default action, meaning patching
    the code is likely to cause strange behaviour for some profiles.

 3. It just feels super dodgy.

[libseccomp-310]: https://github.com/seccomp/libseccomp/issues/310

### What is libseccomp missing? ###

In theory we could implement the behaviour we need purely using libseccomp if
libseccomp supported at least one of the following features:

 * Comparative operators on syscall number (such as "if the syscall number is
   larger than 300, return `-ENOSYS`").

   This would be the most rudimentary feature (and it would probably suffer
   from correctness issues when it comes to inter-architecture semantics --
   since syscall tables can vary wildly in order between architectures) but it
   would allow us to reject system calls outside of the "known" range with
   `-ENOSYS` and handle everything else with `-EPERM`.

 * Per-syscall fallback actions, rather than always using the default action.

   This would allow us to use the explicit `-EPERM` workaround mentioned above,
   and then add a fallback rule for every syscall mentioned in the profile such
   that they will all return `-EPERM`.

 * Multiple comparisons to the same syscall argument in a single rule.

   This would allow us to implement the inverse rules workaround correctly,
   though it would be pretty complicated. Ideally libseccomp would also add
   `MASKED_NEQ` as a comparator to make our lives slightly easier, but the most
   minimal feature we need is to be able to have multiple comparisons against a
   single argument.

 * [Implement the needed `-ENOSYS` behaviour in libseccomp][libseccomp-286],
   such that `-ENOSYS` is automatically treated as a special implicit default
   action if the syscall number is outside of the "minimum kernel version"
   requested.

   This is the nicest solution but the most complicated from libseccomp's point
   of view, though it would require the fewest changes to runc.

[libseccomp-286]: https://github.com/seccomp/libseccomp/issues/286

### License ###

novi-seccomp is licensed under the Apache 2.0 license.

```
Copyright (c) 2021 Aleksa Sarai <cyphar@cyphar.com>
Copyright (c) 2021 SUSE LLC

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
