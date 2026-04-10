import requests
from typing import Optional, Dict, List

# ================== CẤU HÌNH CẦN CHỈNH ==================

GITLAB_BASE_URL = "http://gitlab.gitlabonlinecom.click"
PRIVATE_TOKEN   = "YOUR_PRIVATE_TOKEN"

# Phòng ban (top-level group)
DEPARTMENTS = [
    "Developer",
    "Devsecops",
    "Tester",
    "DB"
]

# Môi trường (subgroup)
ENVS = ["dev", "staging", "prod"]

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
    }
]

# access_level:
# 10 = Guest, 20 = Reporter(RO), 30 = Developer(RW), 40 = Maintainer, 50 = Owner
ACCESS_LEVEL_RW = 30   # Developer => RW
ACCESS_LEVEL_RO = 20   # Reporter => RO

# Phân quyền user: rw/ro theo Dept / Project / Env
# PERMISSIONS[username][dept][project][env] = "rw" hoặc "ro"
PERMISSIONS = {
    "alice": {
        "Developer": {
            "project-a": {
                "dev": "rw",
                "staging": "ro",
                "prod": "ro"
            },
            "project-b": {
                "dev": "rw"
            }
        }
    },
    "bob": {
        "Tester": {
            "test-suite": {
                "dev": "rw",
                "staging": "rw",
                "prod": "ro"
            }
        }
    }
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


# ================== GROUP / SUBGROUP / PROJECT ==================

def ensure_group(name: str, path: str = None, parent_id: Optional[int] = None) -> Dict:
    if path is None:
        path = name.lower().replace(" ", "-")

    # Tìm group theo search để tránh tạo trùng
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
    Tạo cây:
    Dept / Project / Env / (rw, ro)
    + Project GitLab nằm ở level: Dept / Project / Env
    """
    # Lưu mapping để phân quyền sau
    # dept_groups[dept]
    # project_groups[dept][project]
    # env_groups[dept][project][env]
    # perm_groups[dept][project][env]['rw'|'ro']
    dept_groups: Dict[str, Dict] = {}
    project_groups: Dict[str, Dict[str, Dict]] = {}
    env_groups: Dict[str, Dict[str, Dict[str, Dict]]] = {}
    perm_groups: Dict[str, Dict[str, Dict[str, Dict[str, Dict]]]] = {}

    for dept in DEPARTMENTS:
        dept_key = dept
        dept_group = ensure_group(name=dept)
        dept_groups[dept_key] = dept_group
        project_groups[dept_key] = {}
        env_groups[dept_key] = {}
        perm_groups[dept_key] = {}

        dept_id = dept_group["id"]

        projects = PROJECTS_BY_DEPT.get(dept, [])
        for proj in projects:
            proj_key = proj
            # Dept / Project
            proj_group = ensure_group(
                name=proj,
                parent_id=dept_id
            )
            project_groups[dept_key][proj_key] = proj_group
            env_groups[dept_key][proj_key] = {}
            perm_groups[dept_key][proj_key] = {}

            proj_id = proj_group["id"]

            for env in ENVS:
                env_key = env
                # Dept / Project / Env
                env_group = ensure_group(
                    name=env,
                    parent_id=proj_id
                )
                env_groups[dept_key][proj_key][env_key] = env_group

                env_id = env_group["id"]

                # Dept / Project / Env / rw
                rw_group = ensure_group(
                    name="rw",
                    path=f"{env}-rw",
                    parent_id=env_id
                )
                # Dept / Project / Env / ro
                ro_group = ensure_group(
                    name="ro",
                    path=f"{env}-ro",
                    parent_id=env_id
                )

                if env_key not in perm_groups[dept_key][proj_key]:
                    perm_groups[dept_key][proj_key][env_key] = {}
                perm_groups[dept_key][proj_key][env_key]["rw"] = rw_group
                perm_groups[dept_key][proj_key][env_key]["ro"] = ro_group

                # Tạo project GitLab ở level Dept/Project/Env (tùy nhu cầu)
                ensure_project(project_name=proj, namespace_id=env_id)

    return dept_groups, project_groups, env_groups, perm_groups


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


def add_member_to_group(group_id: int, user_id: int, access_level: int):
    try:
        print(f"[PERM] Add user_id={user_id} to group_id={group_id} level={access_level}")
        gitlab_post(f"/groups/{group_id}/members", {
            "user_id": user_id,
            "access_level": access_level
        })
    except requests.HTTPError as e:
        if e.response.status_code == 409:
            print(f"[PERM] User {user_id} already in group {group_id}, skip")
        else:
            raise


def setup_users_and_permissions(perm_groups):
    # 1. Ensure users
    username_to_user: Dict[str, Dict] = {}
    for u in USERS:
        user_obj = ensure_user(u)
        username_to_user[u["username"]] = user_obj

    # 2. Gán quyền theo PERMISSIONS
    for username, dept_map in PERMISSIONS.items():
        user_obj = username_to_user.get(username)
        if not user_obj:
            print(f"[WARN] User {username} not exists, skip")
            continue

        user_id = user_obj["id"]

        for dept, proj_map in dept_map.items():
            if dept not in perm_groups:
                print(f"[WARN] Dept '{dept}' not found, skip")
                continue
            for proj, env_map in proj_map.items():
                if proj not in perm_groups[dept]:
                    print(f"[WARN] Project '{dept}/{proj}' not found, skip")
                    continue
                for env, mode in env_map.items():
                    if env not in perm_groups[dept][proj]:
                        print(f"[WARN] Env '{dept}/{proj}/{env}' not found, skip")
                        continue
                    mode = mode.lower()
                    if mode not in ["rw", "ro"]:
                        print(f"[WARN] Mode '{mode}' invalid, use rw/ro, skip")
                        continue
                    group_obj = perm_groups[dept][proj][env].get(mode)
                    if not group_obj:
                        print(f"[WARN] Group for mode '{mode}' not found, skip")
                        continue

                    access_level = ACCESS_LEVEL_RW if mode == "rw" else ACCESS_LEVEL_RO
                    add_member_to_group(
                        group_id=group_obj["id"],
                        user_id=user_id,
                        access_level=access_level
                    )


# ================== MAIN ==================

def main():
    _, _, _, perm_groups = setup_structure()
    setup_users_and_permissions(perm_groups)
    print("Done.")


if __name__ == "__main__":
    main()
