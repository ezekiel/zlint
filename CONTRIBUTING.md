Contributing Code
-----------------

**Submitting Code for Review.** We strongly prefer multiple small pull
requests (PR), each of which contain a single lint or a small handful of lints,
over a single large PR. This allows for better code review, faster turnaround
times on comments and merging, as well as for contributors to learn from any
requested changes in the initial round of review. We are happy to wait to cut
new a version of ZLint until a set of PRs have been approved and merged.

Adding New Lints
----------------

**Generating Lint Scaffolding.** The scaffolding for a new lints can be created
by running `./newLint.sh <path_name> <lint_name> <structName>`. Path name may be
one of the existing folders under `lints` (for example `apple`, `cabf_br`, `rfc`
etc) and the choice depends on who authors/suggests the lint specification. Lint
names are generally of the form `e_subject_common_name_not_from_san` where the
first letter is one of: `e`, `w`, or `n` (error, warning, or notice respectively).
Struct names following Go conventions, e.g., `subjectCommonNameNotFromSAN`. Example:
`./newLint.sh rfc e_subject_common_name_not_from_san subjectCommonNameNotFromSAN`.
This will generate a new lint in the `lints/rfc` directory with the necessary
fields filled out.

**Choosing a Lint Result Level.** When choosing what `lints.LintStatus` your new
lint should return (e.g. `Notice`,`Warn`, `Error`, or `Fatal`) the following
general guidance may help. `Error` should be used for clear violations of RFC/BR
`MUST` or `MUST NOT` requirements and include strong citations. `Warn` should be
used for violations of RFC/BR `SHOULD` or `SHOULD NOT` requirements and again
should include strong citations. `Notice` should be used for more general "FYI"
statements that violate non-codified community standards or for cases where
citations are unclear. Lastly `Fatal` should be used when there is an
unresolvable error in `zlint`, `zcrypto` or some other part of the certificate
processing.

**Scoping a Lint.** Lints are executed in three steps. First, the ZLint
framework determines whether a certificate falls within the scope of a given
lint by calling `CheckApplies`. This is often used to scope lints to only check
subscriber, intermediate CA, or root CAs. This function commonly calls one of a
select number of helper functions: `IsCA`, `IsSubscriber`, `IsExtInCert`, or
`DNSNamesExist`. Example:

```go
func (l *caCRLSignNotSet) CheckApplies(c *x509.Certificate) bool {
	return c.IsCA && util.IsExtInCert(c, util.KeyUsageOID)
}
```

Next, the framework determines whether the certificate was issued after the
effective date of a Lint by checking whether the certificate was issued prior
to the lint's `EffectiveDate`. You'll also need to fill out the source and
description of what the lint is checking. We encourage you to copy text
directly from the BR or RFC here. Example:

```go
func init() {
	lint.RegisterLint(&lint.Lint{
		Name:          "e_ca_country_name_missing",
		Description:   "Root and Subordinate CA certificates MUST have a countryName present in subject information",
		Citation:      "BRs: 7.1.2.1",
		Source:        lint.CABFBaselineRequirements,
		EffectiveDate: util.CABEffectiveDate,
		Lint:          &caCountryNameMissing{},
	})
}
```

The meat of the lint is contained within the `Execute` function, which is
passed a `x509.Certificate` instance. **Note:** This is an X.509 object from
[ZCrypto](https://github.com/zmap/zcrypto) not the Go standard library. 

Lints should perform their described test and then return a `*LintResult` that
contains a `Status` and optionally a `Details` string, e.g.,
`&LintResult{Status: Pass}`. If you encounter a situation in which you
typically would return a Go `error` object, instead return
`&LintResult{Status: Fatal}`.

Example:

```go
func (l *caCRLSignNotSet) Execute(c *x509.Certificate) *lint.LintResult {
	if c.KeyUsage&x509.KeyUsageCRLSign != 0 {
		return &lint.LintResult{Result: Pass}
	}
	return &lint.LintResult{Result: Error}
}
```

Testing Lints
-------------

**Creating Unit Tests.** Every lint should also have corresponding unit
tests (generally at least one for a success and one for afailure condition). We
have typically generated test certificates using Go (see
[documentation][CreateCertificates] for details), but OpenSSL
could also be used. Test certificates should be placed in `testdata/` and called
from the test file created by `newLint.sh`. You may want to prepend the PEM with
the output of `openssl x509 -text`. You can run your lint against a test
certificate from a unit test using the `test.TestLint` helper function.

[CreateCertificates]: https://golang.org/pkg/crypto/x509/#CreateCertificate 

Example:

```go
func TestBasicConstNotCritical(t *testing.T) {
	inputPath := "caBlankCountry.pem"
	expected := Error
	out := test.TestLint("e_basic_constraints_not_critical", inputPath)
	if out.Status != expected {
		t.Errorf("%s: expected %s, got %s", inputPath, expected, out.Status)
	}
}

```

**Integration Tests.** ZLint's [continuous integration][CI] includes an
integration test phase where all lints are run against a large corpus of
certificates. The number of notice, warning, error and fatal results for each
lint are captured and compared to a set of expected values in a configuration
file. You may need to update these expected values when you add/change lints.
Please see the [integration tests README] for more information.

[CI]: https://travis-ci.org/zmap/zlint
[integration tests README]: https://github.com/zmap/zlint/blob/master/v2/integration/README.md


Updating the TLD Map
--------------------

ZLint maintains [a map of top-level-domains][TLD Map] and their validity periods
that is referenced by linters. This data is updated periodically by a bot
integration using the `zlint-gltd-update` command.

To update the data manually ensure the `zlint-gtld-update` command is installed
and in your `$PATH` and run `go generate`:

	go get github.com/zmap/zlint/cmd/zlint-gtld-update
	go generate github.com/zmap/zlint/...

[TLD Map]: https://github.com/zmap/zlint/blob/master/util/gtld_map.go


Publishing a Release
--------------------

ZLint releases are published via Travis CI using Goreleaser and a bot Github
account. Most of the release process is automated but there is still some manual
effort involved in creating good release notes & communicating news of the
release.

At a high level the release process requires:

1. Preparing release notes.
1. Choosing an appropriate new version per semver.
1. Pushing an annotated release candidate tag.
1. Monitoring CI for successful completion.
1. Editing & Publishing the Github release candidate created by CI.
1. Creating a call-for-testing announcement in Github issues.
1. Emailing the announcement list.
1. Waiting a week.
1. Pushing a final release tag.
1. Editing & Publishing the Github release created by CI.
1. Closing the release announcement Github issue.
1. Emailing the announcement list.

To prepare the release notes examine the diff between `HEAD` and the previous
release tag. E.g. if `v2.0.0` is the latest release, use: 

```bash
git log v2.0.0..HEAD --oneline
```

Try to pull out the commits of importance, following the format of [previous
release notes](https://github.com/zmap/zlint/releases/tag/v2.2.0-rc1). E.g.
pulling out new lints, updated lints, bug fixes, etc. Remember that you don't
need to mention every commit because the release tooling will include a full
change-log of commits. Your job is to emphasize the highlights.

When choosing a new version tag you should reference [the semver
philosophy](http://semver.org/) and the commitments made in the [ZLint
README](https://github.com/zmap/zlint#versioning-and-releases).

Release tags should be annotated with the release notes you prepared so use `-a`
when creating the new tag. You may want to GPG sign the tag, if so add `-s`.
Lastly remember to obey the expected format for the tag name. For final versions
`'v$MAJOR.$MINOR.$PATCH'` and for release candidates
`'v$MAJOR.$MINOR.$PATCH-rc$NUMBER'`. See `git tag` for previous examples to
match.

As an example to create a tag for a first v2.2.0 release candidate run:
```bash
git tag -s -a v2.2.0-rc1
git push origin v2.2.0-rc1
```

After pushing a tag with the expected release format the `deploy` provider
configured in the `.travisci.yml` will kick in and invoke
[Goreleaser](https://goreleaser.com/).

Once the build completes Goreleaser and the `zlintbot` account will have created
a **draft** release in [the project release section of
Github](https://github.com/zmap/zlint/releases). You will need to edit this
release to add your release notes in front of the full change-log of commits. The
release will not be visible until you explicitly publish it. The Goreleaser
automation will attach binary artifacts to the release as they are available.

Now is a good time to create a call-for-testing issue. You can copy a [previous
example](https://github.com/zmap/zlint/issues/466) to create a new one. It
should reference the Github release you just published and is a central place
for folks to report issues with a release candidate.

Next, post to the [ZLint Announcements Mailing
List](https://groups.google.com/forum/#!forum/zlint-announcements). You should
copy the release notes in, link to the Github release, and also reference the
call-for-testing issue.

Assuming the release candidate has no issues that need to be addressed with bug
fixes & a new release candidate tag you can "finalize" the release by pushing
a new tag with the `-rc$NUMBER` portion removed. Repeat the process of editing
the draft Github release to add notes, publishing it, and posting to the mailing
list.

You're done!

For more detail consult the [Goreleaser
docs](https://goreleaser.com/quick-start/), the `deploy` configuration in
[`.travisci.yml`](https://github.com/zmap/zlint/blob/master/.travis.yml#L27-L35),
and the
[`.goreleaser.yml`](https://github.com/zmap/zlint/blob/master/v2/.goreleaser.yml)
project configuration.
