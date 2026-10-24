# Notes

The documentation of the chess repertoire project: how it is designed and why. The code lives in
`../tools`, the repertoires in `../repertoire`; these notes explain what neither says on its own.

Start with [CONTENTS.md](CONTENTS.md), the table of contents.

## What goes here

A note covers one subject: an approach (how memorization is modelled), a part of the software (its
architecture), a piece of infrastructure (the Raspberry Pi, the backups). It says what was chosen, why,
what it costs and what is still open. Details the code states better (signatures, constants) stay in the
code; a note points to the file instead of copying it.

## Conventions

- **English**, like the code, the commits and the studies.
- **One subject per file**, named after the subject in capitals: `MEMORIZATION.md`, `ARCHITECTURE.md`.
  A subject that grows gets a lowercase directory of its own (`memorization/`), its main file keeping the
  name of the subject.
- **Every new file is listed in [CONTENTS.md](CONTENTS.md)**, with one line saying what it answers.
- **Dated decisions.** When a choice is made or changed, write the date next to it (2026-10-23): a later
  finding can overturn it, and the date is what shows the order.
- **Kept current.** A note describes the project as it is; when the code changes what a note says, the note
  changes in the same breath. What was tried and dropped stays, in a short "Dropped" or "History" section,
  so that it is not proposed again.
- **Short.** Ideas and reasons, not tutorials. Numbers carry their source (a command, a date, a data set).
