# Terminal Multiplexer Approaches

## Raw tmux/screen Workflows

The simplest approach: separate tmux windows/panes per Claude Code instance [43] [44].

**Patterns observed:**
- Nested tmux sessions for popup-style interaction [43]
- Multi-window layouts (Claude, tests, logs, git) [44]
- Parallel session spawning scripts [research notes]

**Pros:** Zero dependencies, full terminal control, no overhead.
**Cons:** No state detection, no notifications, manual tab-hopping, easy to miss permission prompts.
**Cross-platform:** Linux, macOS, WSL. No native Windows.

## The "Agentmaxxing" Pattern

Community reports typically running 3-5 parallel sessions [47]. The bottleneck is human attention, not AI capability. This drives demand for monitoring tools.

## GitButler Integration

GitButler's hook integration enables managing parallel sessions without worktrees — changes are tracked as virtual branches [45].

## Gaps and Limitations

- tmux/screen provide no session state awareness
- No visual indicator of which session needs attention
- Cross-desktop switching requires manual effort
- Windows users have no tmux equivalent (WSL is a workaround, not a solution)
