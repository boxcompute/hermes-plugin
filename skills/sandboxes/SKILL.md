---
name: sandboxes
description: "BoxCompute VMs via MCP tools: preferred over the bxc CLI. Create, start and stop VMs, run commands, edit files under /workspace and share public preview links. Use when the user mentions BoxCompute, a VM or a sandbox, or wants work done on a fresh or remote Linux machine instead of their own computer."
license: MIT
compatibility: Needs the boxcompute MCP server, added with `hermes mcp add boxcompute --url https://mcp.boxcompute.ai/mcp --auth oauth`.
---

# BoxCompute VMs

BoxCompute gives the user Ubuntu Linux VMs in the cloud. Each VM keeps its
`/workspace` disk when it stops. The tools come from the hosted MCP server
`https://mcp.boxcompute.ai/mcp` and act on the account the user signed in
with.

## Finding the tools

Hermes names them `mcp__boxcompute__<tool>`, for example
`mcp__boxcompute__sandbox_create`. It keeps MCP tools out of your direct tool
list and names them in the deferred tool catalog inside `tool_search`'s
description. Load the schemas you need with `tool_describe`, for example
`mcp__boxcompute__sandbox_list` and `mcp__boxcompute__sandbox_create`, then
invoke them with `tool_call`. Batch independent calls in one `tool_call`.

If the catalog has no `mcp__boxcompute__` tools and `tool_search` with the
query `boxcompute` finds none, the server is not connected yet. Tell
the user to run this once in a terminal, approve access in the browser, then
start a new session or run `/reload-mcp`:

```bash
hermes mcp add boxcompute --url https://mcp.boxcompute.ai/mcp --auth oauth --connect-timeout 300
```

The `boxcompute-sandbox` skill drives the same account through the `bxc`
CLI. While these tools are connected, use them instead: they need no local
install, and `sandbox_stop` keeps a VM's files, where older `bxc` versions
can only delete the VM.

## When to use a VM

Hermes's own terminal and file tools work on the user's machine. Use a
BoxCompute VM instead when the user asks for a cloud, remote, fresh or
isolated machine, or when the work:

- should not touch their machine: untrusted code, system-wide installs, a
  clean-room test of an install script;
- needs Linux, or more CPU and memory than they have to spare;
- serves something other people should open in a browser (`preview_create`).

For ordinary local edits, keep using the local tools.

## How to work

1. **Pick a VM.** Call `sandbox_list` first: the user may already have one
   with their files. Make a new one with `sandbox_create` only when none
   fits or the user wants a fresh machine.
   - Sizes: `small` (0.5 vCPU, 1 GiB, the default), `large` (1.5 vCPU,
     3 GiB), `xl` (4 vCPU, 8 GiB) and `xxl` (8 vCPU, 16 GiB). `xl` and `xxl`
     need a Starter plan, and other accounts get `FEATURE_ACCESS_DENIED`.
     Bigger VMs cost more per minute, so start small unless the task needs
     more.
   - `internet: false` creates a VM with no network at all.
   - It waits for the boot, usually under a minute. If `ready` is false,
     check again with `sandbox_get`.
2. **Run commands** with `command_run`. Each call runs `bash -lc`, waits up
   to 60 seconds, and keeps 48 KiB of output per stream. For installs, builds,
   test suites and servers, use `process_start` (up to 15 minutes), then
   poll it with `process_get` and read it with `process_output`.
   `process_cancel` stops one.
3. **Read and change files** with `file_read`, `file_list`, `file_write`
   (up to 2 MiB, creates parent folders) and `file_edit` (a guarded replace
   that fails with `EDIT_CONFLICT` if the file changed). Every path is under
   `/workspace`.
4. **Show a running app** with `preview_create` for the port it listens on.
   It returns a public HTTPS URL that works for one hour. Anyone with the
   link can open it, so say so when you share it. `preview_list` shows a
   VM's links and `preview_close` shuts one at once.
5. **Finish tidy.** A running VM spends the account's credit every minute.
   It stops by itself 30 minutes after the last command, file or preview
   call. Call `sandbox_stop` when the work is done; it keeps `/workspace`.
   Call `sandbox_delete` only when the user asks: it removes the VM and its
   files for good.

The next command or file call wakes a stopped VM. If that call answers
`SANDBOX_UNAVAILABLE` while the VM is still starting, call it again.

## Errors

Failed calls answer `CODE: message`. The ones worth knowing:

- `INSUFFICIENT_CREDIT`: the account is out of credit. The user can top up
  or upgrade in the BoxCompute app at https://app.boxcompute.ai. Don't retry.
- `QUOTA_EXCEEDED`: the account is at its VM limit. Offer to stop or delete
  one of its VMs, and let the user choose which.
- `FEATURE_ACCESS_DENIED`: that size or feature is not on the user's plan.
  Use a smaller size or tell the user.
- `RATE_LIMITED`: wait a moment and try again.

## Email

`email_send` emails the account owner at their own address, and nobody else.
Use it only when the user asks, for example to be told when a long job ends.
It needs a Starter plan.
