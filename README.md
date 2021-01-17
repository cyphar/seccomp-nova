## novi-seccomp ##

A container-focused seccomp-cBPF generation library, intended primarily for use
within [runc][runc]. Currently [libseccomp][libseccomp] is missing several
features we need in order to have the behaviour we would like in container
runtimes, so the goal is to have our own cBPF generation code (while still
using libseccomp for all of its handling of syscalls and architecture-specific
semantics).

[runc]: https://github.com/opencontainers/runc
[libseccomp]: https://github.com/seccomp/libseccomp

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
