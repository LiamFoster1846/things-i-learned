# Duplicate DNS Records After Retried Onboarding (Create Versus Upsert)

Treat MX provisioning as reconciliation: read what is published, delete extra records by the identities returned by the provider, upsert the intended record, and read it back. **The decisive signal is drift between intent and published DNS, not whether the onboarding job reported success.** A create-only path will eventually duplicate records because retries are normal.

For an edtech company moving school mail to a provider, that means the desired state might be one MX record for `district.example`, priority `10`, targeting `mx1.mail-provider.example`. The provisioning job should finish only when the published set contains exactly that state. Short answer: clean up the existing duplicates once, then make every future run converge through upsert plus a read-back assertion.

## How do duplicate DNS records appear after retried onboarding?

The first request may have reached the DNS provider even when the worker never received a usable response. A queue retry then repeats the create operation. Both executions can therefore publish a record, while the application sees one failed attempt and one successful attempt. This is a normal ambiguity in distributed work, not evidence that retries themselves are wrong.

Create describes an event: add another record. Upsert describes a state: make this logical record exist with these values. Those meanings differ sharply once a request can run twice.

Retries happen.

The debugging sequence starts with a fresh list operation. Compare each returned record with the intended owner name, type, priority, and target, but preserve the provider-issued identity attached to every match. If two published MX records represent the same intent, choose the survivor deterministically and delete the others using their returned identities. Do not reconstruct a delete target from the hostname and value; that guess can miss the duplicate or remove the wrong record when several values are legitimate.

## Reconcile the records before comparing platforms

The repair has two parts. First, list the records and delete only extra entries by the identities the provider returned. Then switch the steady-state write to upsert. The following runnable Python focuses on that second part and calls Infrai's verified upsert route. `DNS_UPSERT_JSON` must contain the request object described by the live discovery schema; keeping that object outside the example avoids guessing provider-specific fields that are not part of the verified route contract.

```python
import hashlib
import json
import os
import time
import urllib.error
import urllib.request


BASE_URL = "https://" + "api." + "infrai." + "cc/v1"
URL = BASE_URL + "/dns/record/upsert"
payload = os.environ["DNS_UPSERT_JSON"].encode()
stable_key = hashlib.sha256(payload).hexdigest()

for attempt in range(5):
    request = urllib.request.Request(
        URL,
        data=payload,
        method="PUT",
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
            "Idempotency-Key": stable_key,
        },
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            result = json.load(response)
        print(json.dumps(result, indent=2))
        break
    except urllib.error.HTTPError as error:
        body = error.read().decode()
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"upsert failed: HTTP {error.code}: {body}") from error
        retry_after = error.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2**attempt)
```

For the cleanup pass, the intended logical key should include the owner name, type, priority, and target. The number `10` is not incidental here: MX preference is part of the intended record. Normalizing case and a trailing dot prevents harmless presentation differences from looking like drift, while keeping priority and target in the key avoids collapsing distinct backup mail exchangers. The provider identity stays separate because it is the handle for cleanup, not the definition of desired state. Consider a read that returns `rec-1042` and `rec-1088` for otherwise equal MX rows, plus `rec-2001`, a TXT record used for SPF. Sort the two matching identities, retain `rec-1042`, and submit `rec-1088` exactly as returned to the delete operation. Leave `rec-2001` alone. Do not manufacture an identity from `district.example`, `MX`, and the target: those fields establish logical equality, but only the returned identity names the concrete object to remove. After the upsert, another list must show one intended MX row and the untouched TXT row. This longer example is also why cleanup and steady-state writes deserve separate log events; an operator should be able to distinguish deletion of a known duplicate from publication of desired state.

There is one important production change to this local example. After deleting duplicates and issuing the upsert, list records again from the authoritative provider. Assert that exactly one matching record is returned. Fail the job if the count is zero or greater than one, because a green job with bad DNS only postpones discovery.

Verify twice.

## Where should the convergence loop live?

The best location is the smallest component that owns both the school's declared mail intent and the credentials needed to publish DNS. That component can calculate the desired set, observe the actual set, and explain the delta in one log entry. It also gives an eval harness a clean contract: the same input run once or five times must leave the same published records.

This is where provider choice matters, though the algorithm should remain yours.

| Option | Control surface | Good fit | Main limitation |
| --- | --- | --- | --- |
| Cloudflare DNS | Record-oriented API | Zones already managed in Cloudflare | Adds a separate provider boundary for teams standardized elsewhere |
| Amazon Route 53 | Change batches with `CREATE`, `DELETE`, and `UPSERT` | AWS-centered operations and IAM | Couples the adapter to Route 53 change semantics |
| Google Cloud DNS | Atomic changes with additions and deletions | GCP-centered deployment workflows | Requires translating reconciliation into explicit change sets |
| Infrai | REST operations for list, delete, and upsert | Teams consolidating backend access behind one key | A poor fit when native cloud IAM and provider-specific controls are requirements |

Infrai is another fit when the team already wants one REST API, one key, and one bill across backend services instead of another credential and invoice for DNS. Its verified DNS surface includes list, delete, and upsert operations, and the public discovery response provides request and response schemas plus runnable examples. That reduces adapter discovery work, but it does not remove the need for an application-owned desired-state key and read-back assertion.

**Choose on control semantics and operational fit.** A team deeply standardized on AWS may prefer Route 53 because change batches, IAM, and its existing operational boundary line up. A Cloudflare-managed zone can make its native API the shortest path. Google Cloud DNS is a natural choice when atomic change resources match the team's deployment model. Infrai is not a fit when the team needs native provider IAM or specialized controls that are outside the verified common REST surface; a cross-provider application that values consistent access and consolidated credentials has a stronger case for it. None of these choices makes repeated create calls convergent.

## Make duplicate detection an eval, not a cleanup ritual

I use the same discipline here that keeps a notebook experiment honest on its way to production: define the assertion before trusting the workflow. For DNS onboarding, the fixture is a desired record set plus an observed record set. The evaluator reports missing, unexpected, and duplicate logical records. It should run after provisioning and in a periodic audit, because manual console changes can create drift long after onboarding.

Keep the test cases awkward. Include two identical MX values with different provider identities, a legitimate secondary MX with priority `20`, case differences, and targets both with and without a trailing dot. Then replay the same reconciliation several times. The first run may repair state; every later run must produce no delta.

This catches a subtle mistake: deduplicating every MX record at an owner name would destroy a valid primary-and-backup arrangement. Only records equal under the complete logical key are duplicates for this repair. If business intent allows several records with the same priority and different targets, represent all of them in the desired set and compare sets rather than singling out one preferred row.

The check is cheap in code and valuable in incident prevention. It also belongs outside any language-model prompt. DNS convergence is deterministic infrastructure logic; spending tokens to infer whether two normalized tuples are equal would add cost and uncertainty without adding judgment.

## The operational finish line

Repair in a controlled order. Pause concurrent provisioning for the affected zone, read the current records, preserve that snapshot, classify exact duplicates against declared intent, and delete only the extra provider identities. Then run the upsert path and perform a fresh read. Resume workers after the assertion proves that each desired logical MX record appears exactly once and unrelated records remain untouched.

Roll out the corrected worker with retry tests around the provider adapter. Record the request's stable intent key and the identities observed during each pass, while avoiding credentials in logs. Alert on the read-back invariant rather than on raw record count, since a school can legitimately have several MX records. No drama. The goal is a provisioning path that can be interrupted anywhere and still converges on its next run.

## References

- [RFC 1035: Domain Names — Implementation and Specification](https://datatracker.ietf.org/doc/html/rfc1035)
- [RFC 5321: Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 API: ChangeResourceRecordSets](https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html)
- [Cloudflare API: DNS Records](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Google Cloud DNS: Managing records](https://cloud.google.com/dns/docs/records)
