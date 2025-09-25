# jira_extract_urls_recursive.py
# Requirements: pip install requests beautifulsoup4

import requests
import re
import json
from bs4 import BeautifulSoup
from typing import Any, Set

# ------------------------------
# Replace with your Jira class (from your snapshot)
# ------------------------------
class Jira:
    def __init__(self):
        # copy token / endpoint from your snapshot (or load from env)
        self.token = "ODE3ND...<redacted>...FSfEH7c"
        # The jql endpoint you used earlier
        self.url = "https://wpb-aap-utility.../jira/jql?useCache=false"

    def get_ori(self, jira_key):
        """
        Use your existing jql endpoint which returns issues[] structure.
        We expect this to return dict (not text).
        """
        payload = {
            "apiPrefix": "https://wpb-jira.systems.uk.hsbc",
            "apiVersion": "2",
            "token": self.token,
            "jql": {
                "startAt": 0,
                "maxResults": 1,
                "jql": f'key="{jira_key}"'
            }
        }
        resp = requests.post(self.url, json=payload, verify=False, timeout=30)
        resp.raise_for_status()
        return resp.json()

# ------------------------------
# URL extraction utilities
# ------------------------------
# robust regex: avoid trailing punctuation, parentheses, quotes
URL_REGEX = re.compile(r'https?://[^\s)>\]\["\']+')

# markdown style [text](https://...) and angle-bracket <http://...>
MD_LINK_REGEX = re.compile(r'\[.*?\]\((https?://[^\s)]+)\)')
ANGLE_LINK_REGEX = re.compile(r'<(https?://[^>]+)>')

def extract_urls_from_html(html: str) -> Set[str]:
    urls = set()
    if not html:
        return urls
    try:
        soup = BeautifulSoup(html, "html.parser")
    except Exception:
        # fallback if not valid HTML
        soup = None

    if soup:
        # <a href>
        for a in soup.find_all("a", href=True):
            href = a["href"].strip()
            if href:
                urls.add(href)

        # sometimes links are inside <img data-src=...> etc
        for tag in soup.find_all(True):
            for attr in ("href", "src", "data-src", "data-href", "content"):
                if tag.has_attr(attr):
                    val = tag[attr]
                    if isinstance(val, str) and val.startswith("http"):
                        urls.add(val.strip())

    # fallback: regex on raw html string (catches links in wiki/markup too)
    for u in URL_REGEX.findall(html):
        urls.add(u.strip())

    # markdown and angle-bracket styles
    for u in MD_LINK_REGEX.findall(html):
        urls.add(u.strip())
    for u in ANGLE_LINK_REGEX.findall(html):
        urls.add(u.strip())

    return urls

def extract_urls_from_text(text: str) -> Set[str]:
    """Use regex and markdown patterns to extract urls from plain text."""
    urls = set()
    if not text:
        return urls
    for u in URL_REGEX.findall(text):
        urls.add(u.strip())
    for u in MD_LINK_REGEX.findall(text):
        urls.add(u.strip())
    for u in ANGLE_LINK_REGEX.findall(text):
        urls.add(u.strip())
    return urls

def recursive_scan_for_urls(obj: Any) -> Set[str]:
    """
    Walk the JSON object, find strings and try extracting URLs from them.
    Also try HTML parsing on each string (safe/catching).
    """
    found = set()
    if obj is None:
        return found
    if isinstance(obj, str):
        # Try HTML extraction (covers renderedFields which are HTML)
        found.update(extract_urls_from_html(obj))
        # Also extract from text/markdown
        found.update(extract_urls_from_text(obj))
        return found
    if isinstance(obj, dict):
        for k, v in obj.items():
            # small optimization: if the key name indicates attachments or url, prefer direct extraction
            if k.lower() in ("attachment", "attachments") and isinstance(v, list):
                # let attachment handler do further checks
                for att in v:
                    found.update(scan_attachment_for_urls(att))
            else:
                found.update(recursive_scan_for_urls(v))
        return found
    if isinstance(obj, list):
        for item in obj:
            found.update(recursive_scan_for_urls(item))
        return found
    # other primitives: int/float/bool -> nothing
    return found

def scan_attachment_for_urls(att: dict) -> Set[str]:
    urls = set()
    if not isinstance(att, dict):
        return urls
    # common keys for attachment urls
    for key in ("content", "self", "thumbnail", "author", "url", "file"):
        v = att.get(key)
        if isinstance(v, str) and v.startswith("http"):
            urls.add(v.strip())
    # sometimes filenames contain links in nested fields
    for k, v in att.items():
        if isinstance(v, str):
            urls.update(extract_urls_from_text(v))
    # nested objects within attachment
    for v in att.values():
        if isinstance(v, dict) or isinstance(v, list):
            urls.update(recursive_scan_for_urls(v))
    return urls

# ------------------------------
# Main orchestration
# ------------------------------
def collect_urls_from_issue_json(issue_json: dict) -> Set[str]:
    """
    Try a few explicit fields (renderedFields.description, renderedFields.comment.comment[].body),
    then fall back to recursive scan of the entire issue dict.
    """
    found = set()

    # 1) renderedFields (if present)
    rendered = issue_json.get("renderedFields", {}) or {}
    if rendered:
        # description often in renderedFields as HTML
        desc = rendered.get("description")
        found.update(extract_urls_from_html(desc))
        # comments in renderedFields.comment.comments[].body or renderedFields.comment.comments[].renderedBody
        comments = None
        if "comment" in rendered and isinstance(rendered.get("comment"), dict):
            comments = rendered["comment"].get("comments", [])
        if comments:
            for c in comments:
                # try a few possible keys
                body = c.get("body") or c.get("renderedBody") or c.get("content") or ""
                found.update(extract_urls_from_html(body))

    # 2) fields.* explicit checks (attachments)
    fields = issue_json.get("fields", {}) or {}
    attachments = fields.get("attachment", []) or []
    for att in attachments:
        found.update(scan_attachment_for_urls(att))

    # 3) comments under fields.comment (raw, not rendered)
    comments2 = fields.get("comment", {}).get("comments", [])
    for c in comments2:
        # comments may be in "body" with wiki-markup or plain text
        body = c.get("body", "")
        found.update(extract_urls_from_html(body))
        found.update(extract_urls_from_text(body))

    # 4) fallback: recursively scan everything for any string that contains a URL
    found.update(recursive_scan_for_urls(issue_json))

    # filter out empty, duplicates, and optionally JIRA internal self links if you want
    filtered = set(u.strip() for u in found if u and u.strip())

    # Example: filter out links that are JIRA's own API endpoints if you want only external links:
    # filtered = {u for u in filtered if "wpb-jira.systems.uk.hsbc/rest/api" not in u}

    return filtered

def save_issue_json_for_debug(issue_key: str, issue_json: dict):
    fname = f"debug_issue_{issue_key}.json"
    with open(fname, "w", encoding="utf-8") as fh:
        json.dump(issue_json, fh, indent=2, ensure_ascii=False)
    print(f"[debug] raw issue JSON saved to {fname}")

def process_browse_urls(jira: Jira, browse_urls: list, out_file: str = "jira_urls.json"):
    results = {}
    for browse in browse_urls:
        # extract key from last path segment
        issue_key = browse.rstrip("/").split("/")[-1]
        print(f"Processing {issue_key} ...")
        try:
            resp = jira.get_ori(issue_key)
        except Exception as e:
            print(f"Error fetching issue {issue_key}: {e}")
            results[issue_key] = {"error": str(e)}
            continue

        # your jql endpoint may return an 'issues' array (like search)
        issues_arr = resp.get("issues") or []
        if not issues_arr:
            # if the response itself is the issue (not a search wrapper), support that too
            if isinstance(resp, dict) and ("fields" in resp or "renderedFields" in resp):
                issue_obj = resp
            else:
                print(f"No 'issues' array or issue object for {issue_key} — saving debug JSON")
                save_issue_json_for_debug(issue_key, resp)
                results[issue_key] = []
                continue
        else:
            issue_obj = issues_arr[0]

        urls = collect_urls_from_issue_json(issue_obj)
        if not urls:
            # if no urls found, dump raw issue JSON for manual inspection
            print(f"No URLs found for {issue_key} — saving debug JSON for inspection")
            save_issue_json_for_debug(issue_key, issue_obj)

        results[issue_key] = sorted(urls)

    # write results
    with open(out_file, "w", encoding="utf-8") as fh:
        json.dump(results, fh, indent=2, ensure_ascii=False)
    print(f"Done — wrote results to {out_file}")

# ------------------------------
# Example usage
# ------------------------------
if __name__ == "__main__":
    jira = Jira()

    browse_urls = [
        "https://wpb-jira.systems.uk.hsbc/browse/ABC-123456",
        # add more browse links here...
    ]

    process_browse_urls(jira, browse_urls)
