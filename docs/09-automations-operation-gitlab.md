# GitLab Auto Management

Script Python tự động quản lý cấu trúc **Group / Project / User / Permission** trên GitLab.

## 1. Mục tiêu

Tự động hóa:

1. Tạo **group phòng ban** (top-level group)
2. Tạo **project** trong từng group phòng ban  
   → cấu trúc:  
   `Tên phòng ban / Tên project`
3. Tạo / đảm bảo tồn tại **user**
4. Phân quyền:
   - **Maintainer** cho group phòng ban (admin phòng ban)
   - **RW (Developer)** hoặc **RO (Reporter)** cho từng user trên từng project

Lưu ý: Mỗi project là **một repo duy nhất**, bạn tự quản lý 3 branch `dev`, `staging`, `prod` trong Git.  
Script **không** tạo branch, chỉ tạo group/project/user/quyền.

---

## 2. Yêu cầu

- Python 3.x
- Thư viện `requests`:
```bash
  pip install requests
```
    - GitLab self-hosted: http://gitlab.gitlabonlinecom.click
    - Personal Access Token (PAT) trên GitLab với scope: api

---

## 3. Lấy Personal Access Token (PAT)

    1. Đăng nhập http://gitlab.gitlabonlinecom.click

    2. Vào: User menu (avatar) → Preferences

    3. Vào tab Access Tokens (hoặc Personal Access Tokens tuỳ phiên bản)

    4. Tạo token với:
        - Name: api-automation (tùy ý)
        - Scopes: chọn api

    5. Copy token vừa tạo → dán vào biến PRIVATE_TOKEN trong script.

---

## 4. Cấu trúc & Logic phân quyền

### 4.1. Group & Project

Mỗi phòng ban là một group:
   - Developer
   - Devsecops
   - Tester
   - DB
   - 
Mỗi project là một project GitLab nằm dưới group phòng ban tương ứng:

Ví dụ:

   - Developer/project-a
   - Developer/project-b
   - Tester/test-suite
   - DB/db-schema

### 4.2. Phân quyền

Level quyền GitLab:

   - 20 = Reporter → RO
   - 30 = Developer → RW
   - 40 = Maintainer → admin group phòng ban
Script hỗ trợ:

---

## 5. Cấu hình script

Tạo file: gitlab_auto_manage.py với nội dung sau

(chỉ cần chỉnh phần CẤU HÌNH ở đầu file):

```python
import requests
from typing import Optional, Dict, List

# ================== CẤU HÌNH CẦN CHỈNH ==================

GITLAB_BASE_URL = "http://gitlab.gitlabonlinecom.click"
PRIVATE_TOKEN   = "YOUR_PRIVATE_TOKEN"          # Personal Access Token (scope: api)

# Phòng ban (top-level group)
DEPARTMENTS = [
    "Developer",
    "Devsecops",
    "Tester",
    "DB"
]

# Project cho từng phòng ban
PROJECTS_BY_DEPT: Dict[str, List[str]] = {
    "Developer": ["project-a", "project-b"],
    "Devsecops": ["sec-tool"],
    "Tester":    ["test-suite"],
    "DB":        ["db-schema"]
}

# Danh sách user cần tạo/ensure
USERS = [
    {
        "username": "alice",
        "name":     "Alice Developer",
        "email":    "alice@example.com"
    },
    {
        "username": "bob",
        "name":     "Bob Tester",
        "email":    "bob@example.com"
    },
    {
        "username": "devlead",
        "name":     "Developer Lead",
        "email":    "devlead@example.com"
    }
]

# access_level GitLab:
# 20 = Reporter (RO), 30 = Developer (RW), 40 = Maintainer
ACCESS_LEVEL_RW         = 30
ACCESS_LEVEL_RO         = 20
ACCESS_LEVEL_MAINTAINER = 40

# Phân quyền user theo Dept / Project: "rw" hoặc "ro"
# PERMISSIONS[username][department][project] = "rw" | "ro"
PERMISSIONS = {
    "alice": {
        "Developer": {
            "project-a": "rw",
            "project-b": "ro"
        }
    },
    "bob": {
        "Tester": {
            "test-suite": "rw"
        }
    }
}

# Maintainer cho group phòng ban (ngang admin phòng ban)
# DEPT_MAINTAINERS[department] = [list username]
DEPT_MAINTAINERS = {
    "Developer": ["devlead"],
    # "Devsecops": ["sec-lead"],
    # "Tester": ["test-lead"],
    # "DB": ["db-lead"]
}

# ================== HTTP HELPER ==================

session = requests.Session()
session.headers.update({"PRIVATE-TOKEN": PRIVATE_TOKEN})


def gitlab_get(path: str, params: Dict = None):
    r = session.get(f"{GITLAB_BASE_URL}/api/v4{path}", params=params or {})
    r.raise_for_status()
    return r.json()


def gitlab_post(path: str, data: Dict):
    r = session.post(f"{GITLAB_BASE_URL}/api/v4{path}", data=data)
    r.raise_for_status()
    return r.json()


# ================== GROUP & PROJECT ==================

def ensure_group(name: str, path: str = None, parent_id: Optional[int] = None) -> Dict:
    if path is None:
        path = name.lower().replace(" ", "-")

    search_results = gitlab_get("/groups", params={"search": name, "per_page": 100})
    for g in search_results:
        if g["name"] == name and g["path"] == path:
            if (parent_id is None and g.get("parent_id") is None) or \
               (parent_id is not None and g.get("parent_id") == parent_id):
                return g

    data = {"name": name, "path": path}
    if parent_id is not None:
        data["parent_id"] = parent_id

    print(f"[GROUP] Creating group: name={name}, path={path}, parent_id={parent_id}")
    return gitlab_post("/groups", data)


def ensure_project(project_name: str, namespace_id: int) -> Dict:
    search_results = gitlab_get("/projects", params={"search": project_name, "per_page": 100})
    for p in search_results:
        if p["name"] == project_name and p["namespace"]["id"] == namespace_id:
            return p

    data = {
        "name": project_name,
        "namespace_id": namespace_id
    }
    print(f"[PROJECT] Creating project: {project_name} in namespace_id={namespace_id}")
    return gitlab_post("/projects", data)


def setup_structure():
    """
    Tạo cấu trúc:
    Dept (group) / Project (project GitLab)
    """
    dept_groups: Dict[str, Dict] = {}
    projects: Dict[str, Dict[str, Dict]] = {}

    for dept in DEPARTMENTS:
        dept_key = dept
        dept_group = ensure_group(name=dept)
        dept_groups[dept_key] = dept_group
        projects[dept_key] = {}

        dept_id = dept_group["id"]
        proj_list = PROJECTS_BY_DEPT.get(dept, [])

        for proj in proj_list:
            proj_obj = ensure_project(project_name=proj, namespace_id=dept_id)
            projects[dept_key][proj] = proj_obj

    return dept_groups, projects


# ================== USER & PERMISSIONS ==================

def find_user_by_username(username: str) -> Optional[Dict]:
    users = gitlab_get("/users", params={"username": username})
    return users[0] if users else None


def ensure_user(user: Dict) -> Dict:
    existing = find_user_by_username(user["username"])
    if existing:
        return existing

    data = {
        "username": user["username"],
        "name": user["name"],
        "email": user["email"],
        "reset_password": True
    }
    print(f"[USER] Creating user: {user['username']} ({user['email']})")
    return gitlab_post("/users", data)


def add_member_to_project(project_id: int, user_id: int, access_level: int):
    try:
        print(f"[PERM-PROJ] Add user_id={user_id} to project_id={project_id} level={access_level}")
        gitlab_post(f"/projects/{project_id}/members", {
            "user_id": user_id,
            "access_level": access_level
        })
    except requests.HTTPError as e:
        if e.response.status_code == 409:
            print(f"[PERM-PROJ] User {user_id} already in project {project_id}, skip")
        else:
            raise


def add_member_to_group(group_id: int, user_id: int, access_level: int):
    try:
        print(f"[PERM-GRP] Add user_id={user_id} to group_id={group_id} level={access_level}")
        gitlab_post(f"/groups/{group_id}/members", {
            "user_id": user_id,
            "access_level": access_level
        })
    except requests.HTTPError as e:
        if e.response.status_code == 409:
            print(f"[PERM-GRP] User {user_id} already in group {group_id}, skip")
        else:
            raise


def setup_users_and_permissions(dept_groups, projects):
    # 1. Ensure users
    username_to_user: Dict[str, Dict] = {}
    for u in USERS:
        user_obj = ensure_user(u)
        username_to_user[u["username"]] = user_obj

    # 2. Gán Maintainer cho group phòng ban
    for dept, maintainer_usernames in DEPT_MAINTAINERS.items():
        group_obj = dept_groups.get(dept)
        if not group_obj:
            print(f"[WARN] Dept group '{dept}' not found for maintainer assignment")
            continue

        group_id = group_obj["id"]
        for uname in maintainer_usernames:
            user_obj = username_to_user.get(uname)
            if not user_obj:
                print(f"[WARN] Maintainer user '{uname}' not found in USERS, skip")
                continue
            add_member_to_group(
                group_id=group_id,
                user_id=user_obj["id"],
                access_level=ACCESS_LEVEL_MAINTAINER
            )

    # 3. Gán quyền RW/RO cho từng project
    for username, dept_map in PERMISSIONS.items():
        user_obj = username_to_user.get(username)
        if not user_obj:
            print(f"[WARN] User {username} not exists, skip")
            continue

        user_id = user_obj["id"]

        for dept, proj_map in dept_map.items():
            if dept not in projects:
                print(f"[WARN] Dept '{dept}' not found, skip")
                continue
            for proj, mode in proj_map.items():
                if proj not in projects[dept]:
                    print(f"[WARN] Project '{dept}/{proj}' not found, skip")
                    continue

                mode = mode.lower()
                if mode not in ["rw", "ro"]:
                    print(f"[WARN] Mode '{mode}' invalid, use rw/ro, skip")
                    continue

                project_obj = projects[dept][proj]
                project_id = project_obj["id"]
                access_level = ACCESS_LEVEL_RW if mode == "rw" else ACCESS_LEVEL_RO

                add_member_to_project(
                    project_id=project_id,
                    user_id=user_id,
                    access_level=access_level
                )


# ================== MAIN ==================

def main():
    dept_groups, projects = setup_structure()
    setup_users_and_permissions(dept_groups, projects)
    print("Done.")


if __name__ == "__main__":
    main()


```
