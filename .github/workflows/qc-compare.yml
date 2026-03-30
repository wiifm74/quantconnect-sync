import json
import pathlib
import hashlib
import base64
import subprocess
import os
import sys
import time
from typing import Dict, Any, Set, Tuple


INTERNAL_FILES = {".qc-sync-state.json", ".qc-compare-report.json"}


def load_config() -> Tuple[pathlib.Path, Set[str]]:
    config_path = pathlib.Path(".github/qc-sync-config.json")
    if not config_path.exists():
        raise FileNotFoundError("Missing .github/qc-sync-config.json")

    config = json.loads(config_path.read_text(encoding="utf-8"))
    project_dir = pathlib.Path(config["project_dir"])
    allowed_extensions = set(config["allowed_extensions"])
    return project_dir, allowed_extensions


def sha(text: str) -> str:
    return hashlib.sha256(text.encode("utf-8")).hexdigest()


def qc_auth_headers() -> Tuple[str, str]:
    qc_user_id = os.environ["QC_USER_ID"]
    qc_api_token = os.environ["QC_API_TOKEN"]

    timestamp = str(int(time.time()))
    hash_value = hashlib.sha256(f"{qc_api_token}:{timestamp}".encode("utf-8")).hexdigest()
    auth_header = base64.b64encode(f"{qc_user_id}:{hash_value}".encode("utf-8")).decode("utf-8")
    return timestamp, auth_header


def qc_api_post(endpoint: str, payload: Dict[str, Any]) -> Dict[str, Any]:
    timestamp, auth_header = qc_auth_headers()

    result = subprocess.run(
        [
            "curl",
            "-s",
            "-X",
            "POST",
            f"https://www.quantconnect.com/api/v2/{endpoint}",
            "-H",
            "Content-Type: application/json",
            "-H",
            f"Timestamp: {timestamp}",
            "-H",
            f"Authorization: Basic {auth_header}",
            "-d",
            json.dumps(payload),
        ],
        capture_output=True,
        text=True,
        check=False,
    )

    if result.returncode != 0:
        raise RuntimeError(f"curl failed for {endpoint}: {result.stderr}")

    if not result.stdout.strip():
        raise RuntimeError(f"Empty response from QuantConnect endpoint: {endpoint}")

    try:
        data = json.loads(result.stdout)
    except json.JSONDecodeError as e:
        raise RuntimeError(f"Invalid JSON from QuantConnect endpoint {endpoint}: {e}\n{result.stdout}") from e

    return data


def get_project_id() -> int:
    return int(os.environ["PROJECT_ID"])


def fetch_qc_files(allowed_extensions: Set[str]) -> Dict[str, str]:
    data = qc_api_post("files/read", {"projectId": get_project_id()})
    files = data.get("files", [])
    if files is None:
        raise RuntimeError(f"Unexpected response from QuantConnect: {json.dumps(data, indent=2)}")

    qc_files: Dict[str, str] = {}
    for file_obj in files:
        name = file_obj["name"]
        if pathlib.Path(name).suffix not in allowed_extensions:
            continue
        if pathlib.Path(name).name in INTERNAL_FILES:
            continue
        qc_files[name] = file_obj["content"]
    return qc_files


def scan_git_files(project_dir: pathlib.Path, allowed_extensions: Set[str]) -> Dict[str, str]:
    git_files: Dict[str, str] = {}

    if not project_dir.exists():
        return git_files

    for path in project_dir.rglob("*"):
        if not path.is_file():
            continue
        if path.name in INTERNAL_FILES:
            continue
        if path.suffix not in allowed_extensions:
            continue

        rel = path.relative_to(project_dir).as_posix()
        git_files[rel] = path.read_text(encoding="utf-8")

    return git_files


def state_path(project_dir: pathlib.Path) -> pathlib.Path:
    return project_dir / ".qc-sync-state.json"


def report_path(project_dir: pathlib.Path) -> pathlib.Path:
    return project_dir / ".qc-compare-report.json"


def load_state(project_dir: pathlib.Path) -> Dict[str, Any]:
    sp = state_path(project_dir)
    if sp.exists():
        return json.loads(sp.read_text(encoding="utf-8"))
    return {"projectId": str(get_project_id()), "files": {}}


def write_state_from_maps(project_dir: pathlib.Path, git_files: Dict[str, str], qc_files: Dict[str, str]) -> pathlib.Path:
    project_dir.mkdir(parents=True, exist_ok=True)

    all_paths = sorted(set(git_files.keys()) | set(qc_files.keys()))
    files_state = {}

    for rel_path in all_paths:
        git_content = git_files.get(rel_path)
        qc_content = qc_files.get(rel_path)

        files_state[rel_path] = {
            "lastSyncedGitHash": sha(git_content) if git_content is not None else None,
            "lastSyncedQcHash": sha(qc_content) if qc_content is not None else None,
        }

    state = {
        "projectId": str(get_project_id()),
        "files": files_state,
    }

    sp = state_path(project_dir)
    sp.write_text(json.dumps(state, indent=2), encoding="utf-8")
    return sp


def classify(project_dir: pathlib.Path, allowed_extensions: Set[str]) -> Dict[str, Any]:
    qc_files = fetch_qc_files(allowed_extensions)
    git_files = scan_git_files(project_dir, allowed_extensions)
    state = load_state(project_dir)
    state_files = state.get("files", {})

    all_paths = sorted(set(qc_files.keys()) | set(git_files.keys()) | set(state_files.keys()))

    summary = {
        "unchanged": [],
        "git_only_changed": [],
        "qc_only_changed": [],
        "git_deleted": [],
        "qc_deleted": [],
        "conflicts": [],
        "only_in_git": [],
        "only_in_qc": [],
    }

    for rel_path in all_paths:
        git_content = git_files.get(rel_path)
        qc_content = qc_files.get(rel_path)

        git_hash = sha(git_content) if git_content is not None else None
        qc_hash = sha(qc_content) if qc_content is not None else None

        baseline = state_files.get(rel_path, {})
        last_git = baseline.get("lastSyncedGitHash")
        last_qc = baseline.get("lastSyncedQcHash")

        # Exists only in GitHub now
        if git_content is not None and qc_content is None:
            if rel_path in state_files:
                # QC-side deletion since baseline
                if git_hash == last_git:
                    summary["qc_deleted"].append(rel_path)
                else:
                    summary["conflicts"].append(rel_path)
            else:
                summary["only_in_git"].append(rel_path)
            continue

        # Exists only in QC now
        if git_content is None and qc_content is not None:
            if rel_path in state_files:
                # GitHub-side deletion since baseline
                if qc_hash == last_qc:
                    summary["git_deleted"].append(rel_path)
                else:
                    summary["conflicts"].append(rel_path)
            else:
                summary["only_in_qc"].append(rel_path)
            continue

        if git_content is None and qc_content is None:
            continue

        # No baseline yet
        if last_git is None and last_qc is None:
            if git_hash == qc_hash:
                summary["unchanged"].append(rel_path)
            else:
                summary["conflicts"].append(rel_path)
            continue

        # Same content now
        if git_hash == qc_hash:
            summary["unchanged"].append(rel_path)
            continue

        git_changed = (last_git is not None and git_hash != last_git)
        qc_changed = (last_qc is not None and qc_hash != last_qc)

        if git_changed and not qc_changed:
            summary["git_only_changed"].append(rel_path)
        elif qc_changed and not git_changed:
            summary["qc_only_changed"].append(rel_path)
        elif git_changed and qc_changed:
            summary["conflicts"].append(rel_path)
        else:
            summary["conflicts"].append(rel_path)

    return summary


def write_report(project_dir: pathlib.Path, summary: Dict[str, Any]) -> pathlib.Path:
    project_dir.mkdir(parents=True, exist_ok=True)
    rp = report_path(project_dir)
    rp.write_text(json.dumps(summary, indent=2), encoding="utf-8")
    return rp


def ensure_no_conflicts(project_dir: pathlib.Path, allowed_extensions: Set[str]) -> Dict[str, Any]:
    summary = classify(project_dir, allowed_extensions)
    write_report(project_dir, summary)
    if summary["conflicts"]:
        raise RuntimeError(json.dumps(summary, indent=2))
    return summary


def pull_from_qc(project_dir: pathlib.Path, allowed_extensions: Set[str]) -> Dict[str, str]:
    project_dir.mkdir(parents=True, exist_ok=True)
    qc_files = fetch_qc_files(allowed_extensions)

    # Write/create/update files from QC into GitHub working tree
    for rel_path, content in qc_files.items():
        path = project_dir / rel_path
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_text(content, encoding="utf-8")
        print(f"Wrote {path}")

    return qc_files


def push_to_qc(project_dir: pathlib.Path, allowed_extensions: Set[str]) -> Dict[str, str]:
    git_files = scan_git_files(project_dir, allowed_extensions)
    qc_files = fetch_qc_files(allowed_extensions)
    project_id = get_project_id()

    for rel_path, content in sorted(git_files.items()):
        endpoint = "files/update" if rel_path in qc_files else "files/create"
        action = "Updating" if rel_path in qc_files else "Creating"

        print(f"{action} {rel_path}")

        data = qc_api_post(
            endpoint,
            {
                "projectId": project_id,
                "name": rel_path,
                "content": content,
            },
        )
        print(json.dumps(data, indent=2))

    return git_files


def delete_in_qc(rel_path: str) -> None:
    print(f"Deleting in QuantConnect: {rel_path}")
    data = qc_api_post(
        "files/delete",
        {
            "projectId": get_project_id(),
            "name": rel_path,
        },
    )
    print(json.dumps(data, indent=2))


def delete_in_git(project_dir: pathlib.Path, rel_path: str) -> None:
    path = project_dir / rel_path
    if path.exists():
        print(f"Deleting in GitHub working tree: {path}")
        path.unlink()

        # Remove empty parent directories up to project_dir
        parent = path.parent
        while parent != project_dir and parent.exists():
            try:
                parent.rmdir()
            except OSError:
                break
            parent = parent.parent


def main() -> int:
    if len(sys.argv) < 2:
        print("Usage: qc_sync_logic.py [compare|check|pull|push|write-state]")
        return 1

    command = sys.argv[1]
    project_dir, allowed_extensions = load_config()

    print(f"Using project id: {get_project_id()}")
    print(f"Project dir: {project_dir}")
    print(f"Allowed extensions: {sorted(allowed_extensions)}")

    try:
        if command == "compare":
            summary = classify(project_dir, allowed_extensions)
            write_report(project_dir, summary)
            print(json.dumps(summary, indent=2))
            return 2 if summary["conflicts"] else 0

        if command == "check":
            summary = ensure_no_conflicts(project_dir, allowed_extensions)
            print(json.dumps(summary, indent=2))
            return 0

        if command == "pull":
            summary = ensure_no_conflicts(project_dir, allowed_extensions)

            # QC is authoritative for this run, so QC-side deletions should propagate into GitHub.
            for rel_path in summary["qc_deleted"]:
                delete_in_git(project_dir, rel_path)

            qc_files = pull_from_qc(project_dir, allowed_extensions)
            sp = write_state_from_maps(project_dir, qc_files, qc_files)
            print(f"Wrote {sp}")
            return 0

        if command == "push":
            summary = ensure_no_conflicts(project_dir, allowed_extensions)

            # Safe QC-side deletions should propagate when GitHub is authoritative for this run.
            for rel_path in summary["git_deleted"]:
                delete_in_qc(rel_path)

            git_files = push_to_qc(project_dir, allowed_extensions)

            # Re-read QC after push because QC may normalize or rewrite files
            qc_files = fetch_qc_files(allowed_extensions)

            sp = write_state_from_maps(project_dir, git_files, qc_files)
            print(f"Wrote {sp}")
            return 0

        if command == "write-state":
            git_files = scan_git_files(project_dir, allowed_extensions)
            sp = write_state_from_maps(project_dir, git_files, git_files)
            print(f"Wrote {sp}")
            return 0

        print(f"Unknown command: {command}")
        return 1

    except Exception as e:
        print(str(e))
        return 1


if __name__ == "__main__":
    raise SystemExit(main())
