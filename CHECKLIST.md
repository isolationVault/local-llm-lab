<!-- Visibility: public -->
# Pre-publish checklist

Check every box before an entry goes into `entries/`.

## Privacy
- [ ] No hostnames, machine names or profile names
- [ ] No usernames or local paths (use `<runner-dir>` and `<models-dir>`)
- [ ] No IP addresses or private network names
- [ ] Test ports and run labels are generic (use `<port>`)
- [ ] No screenshots without review
- [ ] Scan passed, and it reported how many identifiers it loaded

## Accuracy
- [ ] Numbers checked against raw output
- [ ] Grades checked against the written key
- [ ] Units stated and consistent (decimal GB or GiB, not mixed)
- [ ] Runner build copied from the runner's own version output
- [ ] Sampling stated, and any difference from the model card noted
- [ ] Harness and thinking mode stated
- [ ] Anything not measured says "not recorded"

## Credit and licenses
- [ ] Model license line present, with a link and the date checked
- [ ] Attribution line included if the model's maker asks for one
- [ ] All links open

## Final
- [ ] Status stamp present
- [ ] Owner approved
