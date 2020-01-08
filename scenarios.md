# Origin isolation scenarios

This document considers various combinations of documents being loaded, with different relationships to each other and different origin isolation signals, to illustrate the consequences of the [proposed algorithm](./README.md#specification-plan).

Notes:

* We use `https://e.com` as our example domain for brevity. "e" stands for "example" but is short enough to fit into tables.
* All of these considerations apply for popups, not just frames, in the same ways.
* All of these considerations apply for workers, not just documents, in the same ways.
* We use the notation `Origin{}` and `Site{}` to denote the type of agent cluster key.

## Two-document scenarios

| Main frame              | Subframe                 | Result                                                               |
| ------------------------|--------------------------|----------------------------------------------------------------------|
| `https://e.com` w/ OI   | `https://e.com` w/ OI    | Both in `Origin{https://e.com}`                                      |
| `https://e.com` w/ OI   | `https://e.com` w/o OI   | Both in `Origin{https://e.com}`                                      |
| `https://e.com` w/o OI  | `https://e.com` w/o OI   | Both in `Site{https://e.com}`                                        |
| `https://e.com` w/o OI  | `https://e.com` w/ OI    | Both in `Site{https://e.com}`                                        |
| `https://e.com` w/o OI  | `https://x.e.com` w/ OI  | Main in `Site{https://e.com}`   <br>Sub in `Origin{https://x.e.com}` |
| `https://e.com` w/o OI  | `https://x.e.com` w/ OI  | Main in `Site{https://e.com}`   <br>Sub in `Origin{https://x.e.com}` |
| `https://e.com` w/ OI   | `https://x.e.com` w/ OI  | Main in `Site{https://e.com}`   <br>Sub in `Origin{https://x.e.com}` |
| `https://e.com` w/ OI   | `https://x.e.com` w/o OI | Main in `Origin{https://e.com}` <br>Sub in `Site{https://e.com}`     |

## Three-document scenarios

| Main frame                 | Subframe A                  | Subframe B                 | Result                                                               |
| ---------------------------|-----------------------------|----------------------------|----------------------------------------------------------|
| `https://e.com`<br>w/ OI   | `https://x.e.com`<br>w/o OI | `https://x.e.com`<br>w/ OI | Main in `Origin{https://e.com}`<br>_If A loads first_: A and B both in `Site{https://e.com}` <br>If _B loads first_: A and B both in `Origin{https://x.e.com}` |
| `https://e.com`<br>w/o OI  | `https://x.e.com`<br>w/o OI | `https://x.e.com`<br>w/ OI | Main in `Site{https://e.com}`<br>_If A loads first_: A and B both in `Site{https://e.com}` <br>If _B loads first_: A and B both in `Origin{https://x.e.com}` |
| `https://e.com`<br>w/ OI   | `https://a.e.com`<br>w/o OI | `https://b.e.com`<br>w/ OI | Main in `Origin{https://e.com}`<br>A in `Site{https://e.com}`<br>B in `Origin{https://b.e.com}` |
| `https://e.com`<br>w/o OI  | `https://a.e.com`<br>w/o OI | `https://b.e.com`<br>w/ OI | Main and A in `Origin{https://e.com}`<br>B in `Origin{https://b.e.com}` |
