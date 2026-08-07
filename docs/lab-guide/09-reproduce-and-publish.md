# Module 09 — Reproduce and Publish

## What you will learn

- How to prepare a public security project without leaking sensitive data.
- How to prove that another learner can clone and navigate the repository.
- How to separate completed evidence from planned work.

## 1. Review the repository status honestly

Keep a workstream marked **planned**, **build pending**, or **evidence pending**
until its acceptance criteria and public evidence exist. Documentation alone is
not proof that a detection or playbook was executed.

## 2. Confirm excluded data

The project `.gitignore` excludes common secrets and raw evidence, but it is not
a security boundary. Before staging files, review names and contents manually.

Do not publish:

- `.env` files, credentials, API tokens, certificates, or private keys.
- Splunk license or authentication material.
- Raw EVTX, PCAP, crash dumps, or unrestricted diagnostic logs.
- Personal usernames, email addresses, unrelated IPs, or browser sessions.
- Real malware, destructive payloads, or offensive infrastructure.

## 3. Inspect changes before committing

From a local clone:

```bash
git status --short
git diff -- README.md docs/ .gitignore
```

- `git status --short` gives a compact list of changed/untracked files.
- `git diff --` shows unstaged content only for the named paths.

After selecting intended files:

```bash
git add README.md .gitignore docs/
git diff --cached --stat
git diff --cached
```

- `git add` stages only the project documentation paths.
- `--cached --stat` summarizes the staged change.
- `git diff --cached` displays the exact staged content.

Do not use `git add -A` in a mixed workspace unless every change belongs to
this project.

## 4. Scan for obvious sensitive patterns

Review likely terms before commit:

```bash
git grep -n -I -E 'password|passwd|secret|token|api[_-]?key|BEGIN.*PRIVATE KEY'
```

- `git grep` searches tracked content.
- `-n` includes line numbers.
- `-I` skips binary files.
- `-E` enables an extended regular expression.

Matches are not automatically leaks; documentation may use these words. Review
every result. A clean result also does not guarantee that no secret exists.

## 5. Commit with a meaningful message

```bash
git commit -m "docs: add validated SOC lab evidence"
```

Use the actual scope of the change. Do not claim validation in the message when
only planning documents changed.

## 6. Push your branch or approved default branch

```bash
git push origin main
```

- `origin` is the remote created by the clone.
- `main` is the branch being published.

Contributors should normally push a feature branch and open a pull request.
Repository owners may publish directly only when that matches their workflow.

## 7. Verify links and diagrams on GitHub

Open the public repository and inspect:

- README table of contents.
- Mermaid architecture diagrams.
- Every `docs/lab-guide/` and `docs/reference/` link.
- Evidence image paths.
- Code blocks and tables on desktop and narrow widths.
- Project status language.

A local Markdown preview does not prove GitHub rendered the same result.

## 8. Perform a clean-clone test

Choose a new temporary parent directory:

```bash
git clone https://github.com/RanvirSinghSaini/soc-monitoring-incident-response.git
cd soc-monitoring-incident-response
git status --short --branch
git remote -v
```

Expected results:

- Clone completes without authentication because the repository is public.
- Current branch tracks `origin/main`.
- Working tree has no changes.
- Remote URL matches the public repository.

Confirm required paths:

```bash
test -f README.md
test -f docs/lab-guide/README.md
test -f docs/reference/COMMAND-REFERENCE.md
test -f docs/evidence/README.md
```

On Bash, `test -f` exits successfully when the file exists. It normally prints
nothing; use the shell exit status to determine success.

## 9. Complete a reproduction report

Another learner should record:

- Clone date and commit SHA.
- Host and VM resources.
- Operating-system and tool versions.
- Address substitutions.
- Modules completed.
- Acceptance tests passed/failed.
- Detection/test-case IDs.
- Sanitized evidence references.
- Documentation corrections.

## 10. Mark work complete only after verification

Change project status only when:

- The fresh clone passes.
- Required evidence paths resolve.
- Every claimed detection has recorded tests.
- No known secret or sensitive raw evidence is committed.
- The final case report and lessons learned exist.

## Pass criteria

- [ ] Staged content was reviewed line by line.
- [ ] Sensitive-pattern matches were reviewed.
- [ ] GitHub links and diagrams render.
- [ ] A new public clone succeeds.
- [ ] Required files exist in the clone.
- [ ] Status matches the available evidence.
- [ ] Reproduction report records the tested commit SHA.

## Official references

- [GitHub: cloning a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/cloning-a-repository)
- [GitHub: about remote repositories](https://docs.github.com/en/get-started/git-basics/about-remote-repositories)
- [GitHub: removing sensitive data](https://docs.github.com/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
