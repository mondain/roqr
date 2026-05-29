# RTMP over QUIC

This is the working area for the individual Internet-Draft, "RTMP over QUIC".

* [Datatracker Page](https://datatracker.ietf.org/doc/draft-gregoire-rtmp-over-quic)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-gregoire-rtmp-over-quic)

## Draft References

Draft text references used while editing this specification are kept under
`docs/`. Review those text files first when comparing this draft to related
drafts.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

Contributions can be made by creating pull requests. The GitHub interface
supports creating pull requests using the Edit button.

## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed. See
[the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

## Submission Checklist

Run the practical pre-submit checks:

```sh
$ make check-submission
```

This runs:

```sh
$ make latest lint
$ make idnits idnits_mode=forgive-checklist
```

Expected residual warnings that can be tooling/environment related:

* `POSSIBLE_DOWNREF` for active Internet-Draft references when status lookup fails.
* `LINE_PI` from generated `<?line ...?>` processing instructions in markdown-to-XML workflows.
* `INVALID_REFERENCES_NAME` when generated XML wraps normative and informative
  references in a combined `References` section before the actual
  `Normative References` and `Informative References` subsections.
* `NON_ASCII_UTF8` as a review reminder. A source scan of the repo-authored
  draft and support files should show whether any non-ASCII text was actually
  introduced.

For a printed checklist in terminal:

```sh
$ make submission-checklist
```
