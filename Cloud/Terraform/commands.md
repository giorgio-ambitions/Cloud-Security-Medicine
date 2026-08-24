Better review

⭐ Overall assessment

The code is generally well structured for a CLI entry point and follows Terraform's existing architecture. The command registration is explicit and easy to scan, and the shared command.Meta object keeps command initialization consistent.

I would not recommend a large refactor of initCommands simply to reduce its size. Since this function is effectively the central registry for Terraform's CLI commands, keeping command registration together has a practical benefit.

There are, however, a few improvements worth considering.

1. Improve test coverage around command registration

This is the strongest improvement.

The function establishes important invariants:

primary commands must exist in Commands
hidden commands shouldn't accidentally appear in the primary list
experimental commands should only exist when experiments are enabled
legacy commands such as env should continue to work
command factories should return the expected command types

Rather than refactoring the registration code, add tests around those invariants.

For example:

func TestInitCommands_PrimaryCommandsExist(t *testing.T) {
    // initialize commands...

    for _, name := range PrimaryCommands {
        if _, ok := Commands[name]; !ok {
            t.Errorf("primary command %q is not registered", name)
        }
    }
}

This gives you protection against accidentally breaking the CLI when commands are added or removed.

2. Test experimental command registration

This section:

if meta.AllowExperimentalFeatures {
    Commands["cloud"] = func() (cli.Command, error) {
        return &command.CloudCommand{
            Meta: meta,
        }, nil
    }
}

has behavior worth testing.

Specifically:

experiments disabled → "cloud" is unavailable
experiments enabled  → "cloud" is registered

That is more valuable than extracting the block into another function merely for organizational reasons.

3. Preserve the existing host-validation contract

I would not change this:

host, err := svchost.ForComparison(userHost)
if err != nil {
    continue
}

The surrounding comment explicitly establishes that configuration is expected to have been validated earlier.

If this behavior is considered risky, the appropriate improvement would be to verify that the earlier validation guarantees are covered by tests—not to add a warning here.

A better review comment would therefore be:

The current behavior intentionally ignores invalid hosts because configuration validation is expected to have occurred earlier. It would be useful to have a test covering that validation contract so future changes don't accidentally make this assumption invalid.

4. Consider documenting the lifetime of makeShutdownCh

The signal handling is appropriate for a CLI, so I wouldn't call this a goroutine leak.

However, the goroutine has process-lifetime semantics:

go func() {
    for {
        <-signalCh
        resultCh <- struct{}{}
    }
}()

A small improvement would be to make that lifecycle explicit in a comment or test its expected behavior.

For example, a test could verify that each received signal produces a shutdown notification.

I would not introduce context cancellation solely to "fix a leak" unless there is an actual lifecycle problem.

5. Be cautious about changing the command factories

The repeated pattern:

"plan": func() (cli.Command, error) {
    return &command.PlanCommand{
        Meta: meta,
    }, nil
},

looks repetitive, but it is also extremely readable.

I would not introduce a generic wrap() helper just to eliminate those few lines.

The existing code makes it immediately obvious:

"plan" → PlanCommand

That's valuable in a CLI registry.

If command construction eventually becomes more complicated, a helper or table-driven approach could make sense, but there isn't enough evidence here to justify it now.

6. Keep the global command registry unless there's a concrete testing problem

The review's suggestion to replace:

var Commands map[string]cli.CommandFactory

with:

type CommandRegistry struct {
    Commands        map[string]cli.CommandFactory
    PrimaryCommands []string
    HiddenCommands  map[string]struct{}
}

is architecturally interesting, but it's a much larger change.

It would affect the surrounding CLI architecture and isn't necessary merely because global state is generally considered undesirable.

I'd only propose this if you can demonstrate a concrete problem—for example, tests interfering with each other because of global state.

The improvements I'd actually submit

If this were a PR review, I'd focus on these:

Priority 1 — Add registration tests

Verify:

PrimaryCommands ⊆ Commands
HiddenCommands ∩ PrimaryCommands = ∅
experimental commands respect AllowExperimentalFeatures
legacy commands remain registered

Priority 2 — Test signal behavior

Verify that an interrupt produces a shutdown notification and that multiple interrupts are handled correctly.

Priority 3 — Preserve and test configuration validation

Don't change the continue; instead, make sure the validation contract that precedes initCommands is tested.

Priority 4 — Leave the explicit command factories alone

The duplication is acceptable because it improves discoverability and keeps each command's construction obvious.
