# CSV Inspection & Planning Summary

Source file: `input/leadinfo - 200+.csv`

## Key findings
- Row count: 469 data rows (excluding header).
- Header columns: 47 total.
- Most likely domain column: `CompanyDomain` (100% non-empty, domain-like values).
- Company name column present: `Company Name` (467 non-empty rows).
- Domain quality:
  - Empty domains: 0
  - Duplicate normalized domains: 9 distinct duplicated domains
  - Malformed domains: 3 (`aryzta.com:443`, `peab.se:443`, `firstcamp.se:443`) due to port suffix
  - General format is otherwise clean (no scheme/path/whitespace in `CompanyDomain`).

## Recommended preparation
1. Read CSV in immutable mode and keep original untouched.
2. Create derived working dataset with:
   - `input_domain_raw` = original `CompanyDomain`
   - `input_domain` = lowercase, trim, strip scheme/path/www, strip port, punycode normalize
3. Validate domains via strict regex + public suffix sanity checks.
4. Flag records:
   - `domain_empty`
   - `domain_malformed`
   - `domain_duplicate`
5. Build unique-domain queue for discovery while keeping mapping back to all original rows.

## Proposed output schema (contact-level)
- input_domain
- matched_company_name
- contact_rank
- full_name
- first_name
- last_name
- job_title
- persona_bucket
- seniority
- linkedin_url
- apollo_person_id
- apollo_organization_id
- city
- country
- confidence_score
- match_reason
- source
- search_layer_used
- status
- notes

## Waterfall (future implementation)
For each normalized domain:
1. Search Workplace titles.
2. If <3 accepted contacts, search Facilities/Building/Office/Real Estate titles.
3. If <3 accepted contacts, search IT fallback titles.
4. Deduplicate across layers by person id / linkedin / normalized name+company.
5. Reject irrelevant roles using exclusion rules.
6. Stop at 3 accepted contacts.

## Ranking/scoring (future implementation)
Composite score (0-100):
- Title relevance to persona (0-45)
- Seniority fit (0-20)
- Current-role/company match confidence (0-20)
- Data completeness (linkedin/person id/city/country) (0-10)
- Layer penalty for fallback searches (0 to -10)

## Error handling/logging (future implementation)
- Structured JSONL logs with run_id, domain, layer, query_signature, status, duration_ms, error_code.
- Per-domain status ledger: `ok`, `partial`, `no_match`, `invalid_domain`, `error`.
- Retry policy only for transient transport errors (not logical no-match).
- Save checkpoints after each domain batch.
- Sample mode hard-cap: first 25 unique domains.

## Next implementation step
Build a **preflight + sample-run skeleton** only:
1. Loader/validator for input CSV and domain normalization.
2. Unique-domain queue + duplicate mapping.
3. Sample-mode selector (first 25 unique domains).
4. Output writer using final schema (empty placeholder rows allowed for dry-run).
5. Logging framework and status ledger.
6. Config file for persona dictionaries, exclusions, and waterfall layers.
7. No Apollo calls yet.
