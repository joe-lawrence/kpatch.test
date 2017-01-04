kpatch Patch Author Guide
=========================

Because kpatch-build is relatively easy to use, it can be easy to assume that a
successful patch module build means that the patch is safe to apply.  But in
fact that's a very dangerous assumption.

There are many pitfalls that can be encountered when creating a live patch.
This document attempts to guide the patch creation process.  It's a work in
progress.  If you find it useful, please contribute!

Patch Analysis
--------------

kpatch provides _some_ guarantees, but it does not guarantee that all patches
are safe to apply.  Every patch must also be analyzed in-depth by a human.

The most important point here cannot be stressed enough.  Here comes the bold:

**Do not blindly apply patches.  There is no subsitute for human analysis and
reasoning on a per-patch basis.  All patches must be thoroughly analyzed by a
human kernel expert who completely understands the patch and the affected code
and how they relate to the live patching environment.**

kpatch vs livepatch vs kGraft
-----------------------------

This document assumes that the kpatch core module is being used.  Other live
patching systems (e.g., livepatch and kGraft) have different consistency
models.  Each comes with its own guarantees, and there are some subtle
differences.  The guidance in this document applies **only** to kpatch.

Patch upgrades
--------------

Due to potential unexpected interactions between patches, it's highly
recommended that when patching a system which has already been patched, the
second patch should be a cumulative upgrade which is a superset of the first
patch.

Data structure changes
----------------------

kpatch patches functions, not data.  If the original patch involves a change to
a data structure, the patch will require some rework, as changes to data
structures are not allowed by default.

Usually you have to get creative.  There are several possible ways to handle
this:

### Change the code which uses the data structure

Sometimes, instead of changing the data structure itself, you can change the
code which uses it.

For example, consider this
[patch](http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=54a20552e1eae07aa240fa370a0293e006b5faed).
which has the following hunk:

```
@@ -3270,6 +3277,7 @@ static int (*const svm_exit_handlers[])(struct vcpu_svm *svm) = {
 	[SVM_EXIT_EXCP_BASE + PF_VECTOR]	= pf_interception,
 	[SVM_EXIT_EXCP_BASE + NM_VECTOR]	= nm_interception,
 	[SVM_EXIT_EXCP_BASE + MC_VECTOR]	= mc_interception,
+	[SVM_EXIT_EXCP_BASE + AC_VECTOR]	= ac_interception,
 	[SVM_EXIT_INTR]				= intr_interception,
 	[SVM_EXIT_NMI]				= nmi_interception,
 	[SVM_EXIT_SMI]				= nop_on_interception,
```

`svm_exit_handlers[]` is an array of function pointers.  The patch adds a
`ac_interception` function pointer to the array at index `[SVM_EXIT_EXCP_BASE +
AC_VECTOR]`.  That change is incompatible with kpatch.

Looking at the source file, we can see that this function pointer is only
accessed by a single function, `handle_exit()`:

```
        if (exit_code >= ARRAY_SIZE(svm_exit_handlers)
            || !svm_exit_handlers[exit_code]) {
                WARN_ONCE(1, "svm: unexpected exit reason 0x%x\n", exit_code);
                kvm_queue_exception(vcpu, UD_VECTOR);
                return 1;
        }

        return svm_exit_handlers[exit_code](svm);
```

So an easy solution here is to just change the code to manually check for the
new case before looking in the data structure:

```
@@ -3580,6 +3580,9 @@ static int handle_exit(struct kvm_vcpu *vcpu)
                return 1;
        }

+       if (exit_code == SVM_EXIT_EXCP_BASE + AC_VECTOR)
+               return ac_interception(svm);
+
        return svm_exit_handlers[exit_code](svm);
 }
```

Not only is this an easy solution, it's also safer than touching data since
kpatch creates a barrier between the calling of old functions and new
functions.

### Use a kpatch load hook

If you need to change the contents of an existing variable in-place, you can
use the KPATCH_LOAD_HOOK macro to specify a function to be called when the
patch module is loaded.

Don't forget to protect access to the data as needed.

Also be careful when upgrading.  If patch A has a load hook which writes to X,
and then you load patch B which is a superset of A, in some cases you may want
to prevent patch B from writing to X, if A is already loaded.

Examples needed.

### Use a shadow variable

If you need to add a field to an existing data structure, or even many existing
data structures, you can use the `kpatch_shadow_*()` functions.

Example needed (see shadow-newpid.patch in the integration tests directory).

Data semantic changes
---------------------

Sometimes, the data itself remains the same, but how it's used is changed.  A
common example is locking semantic changes.

Example needed.

Init code changes
-----------------

Any code which runs in an `__init` function or during module or device
initialization is problematic, as it may have already run before the patch was
applied.  The patch may require a load hook function which detects whether such
init code has run, and which rewrites or changes the original initialization to
force it into the desired state.  Some changes involving hardware init are
inherently incompatible with live patching.

Header file changes
-------------------

When changing header files, be extra careful.  If data is being changed, you
probably need to modify the patch.  See "Data struct changes" above.

If a function prototype is being changed, make sure it's not an exported
function.  Otherwise it could break out-of-tree modules.

Many header file changes result in a complete rebuild of the kernel tree, which
makes kpatch-build have to compare every .o file in the kernel.  It slows the
build down a lot, and can even fail to build if kpatch-build has any bugs
lurking.  If it's a trivial header file change, like adding a macro, it's
advisable to just move that macro into the .c file where it's needed to avoid
changing the header file at all.

Dealing with unexpected changed functions
-----------------------------------------

In general, it's best to patch as minimally as possible.  If kpatch-build is
reporting some unexpected function changes, it's always a good idea to try to
figure out why it thinks they changed.  In many cases you can change the source
patch so that they no longer change.

Some examples:

* If a changed function was inlined, then the callers which inlined the
  function will also change.  In this case there's nothing you can do to
  prevent the extra changes.

* If your patch adds a call to a function where the original version of the
  function's ELF symbol has a .constprop or .isra suffix, and the corresponding
  patched function doesn't, that means the patch caused gcc to no longer
  perform an interprocedural optimization, which affects the function and all
  its callers.  If you want to prevent this from happening, copy/paste the
  function with a new name and call the new function from your patch.

Removing references to static local variables
---------------------------------------------

Removing references to static locals will fail to patch unless extra steps are taken.
Static locals are basically global variables because they outlive the function's
scope. They need to be correlated so that the new function will use the old static
local. That way patching the function doesn't inadvertently reset the variable
to zero; instead the variable keeps its old value.

To work around this limitation one needs to retain the reference to the static local.
This might be as simple as adding the variable back in the patched function in a 
non-functional way and ensuring the compiler doesn't optimize it away.
