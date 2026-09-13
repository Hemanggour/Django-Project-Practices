# Django Project Practices & Conventions

> **Source:** Sound-Node (`BACKEND/`)
> **Purpose:** This document captures the Django + DRF engineering conventions used in this project. Use it as a reference to build new modules, endpoints, or entirely new Django projects with the same style.
>
> **Stack:** Django 5, Django REST Framework, PostgreSQL, SimpleJWT (cookie-based), S3/MinIO object storage via django-storages.

---

## Table of Contents

1. [Project Layout](#1-project-layout)
2. [Settings & Configuration](#2-settings--configuration)
3. [App Structure](#3-app-structure)
4. [Models](#4-models)
5. [Services Layer](#5-services-layer)
6. [Serializers](#6-serializers)
7. [Views (APIView pattern)](#7-views-apiview-pattern)
8. [Authentication](#8-authentication)
9. [Response Structure & Formatting](#9-response-structure--formatting)
10. [URL Design](#10-url-design)
11. [Signals](#11-signals)
12. [Admin](#12-admin)
13. [File Storage Abstraction](#13-file-storage-abstraction)
14. [Testing](#14-testing)
15. [Deployment (Docker / Gunicorn)](#15-deployment-docker--gunicorn)
16. [New App / Endpoint Checklist](#16-new-app--endpoint-checklist)

---

## 1. Project Layout

```
project/
├── BACKEND/                    # Django project root
│   ├── project/                # Core config app
│   │   ├── settings.py
│   │   ├── urls.py
│   │   ├── asgi.py
│   │   └── wsgi.py
│   ├── account/                # Auth & user management
│   ├── music/                  # Business domain (media)
│   │   └── services/           # Business logic layer
│   ├── utils/                  # Cross-app shared utilities
│   │   ├── response_wrapper.py
│   │   └── storage.py
│   ├── manage.py
│   ├── entrypoint.sh           # Container startup steps
│   └── wait_for_db.py
├── FRONTEND/                   # SPA (consumes /api)
├── nginx/
├── docker-compose.yml
└── docs/
```

**Rule:** One folder per domain app, a `project/` config package, and a top-level
`utils/` package for code shared across apps (response helpers, storage backends).
Business logic that isn't a simple ORM call lives in `<app>/services/`.

---

## 2. Settings & Configuration

**12-factor / environment-driven settings.**

- All secrets and environment-specific values come from environment variables.
  - `python-dotenv` loads a `.env` file via `load_dotenv()`.
  - Access via `os.getenv("KEY", default)` with safe defaults.
  - `SECRET_KEY = os.getenv("SECRET_KEY", None)`
- The `.env.example` file documents every variable with comments.
- `DEBUG = os.getenv("DEBUG", "False").lower() == "true"` — parse strings to bools.
- Lists are parsed from comma-separated env strings:
  `ALLOWED_HOSTS = os.getenv("ALLOWED_HOSTS", "").split(",")`.
- Database configured via `dj_database_url.config(default=os.getenv("DATABASE_URL"))`
  (single `DATABASE_URL` string powers the DB connection).
- `AUTH_USER_MODEL = "account.User"` — custom user set globally.
- `DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"`.

### DRF global defaults (`REST_FRAMEWORK`)

```python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
        "rest_framework.authentication.SessionAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
    "DEFAULT_PAGINATION_CLASS": "utils.response_wrapper.StandardPagination",
    "PAGE_SIZE": 10,
}
```

- Global default permission is `IsAuthenticated`; public views override per-view.
- Global pagination class is reused across every list endpoint.

### SimpleJWT

```python
SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(days=1),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": False,
    "BLACKLIST_AFTER_ROTATION": True,
    "AUTH_HEADER_TYPES": ("Bearer",),
    "USER_ID_FIELD": "user_uuid",    # JWT subject is the external uuid
    "USER_ID_CLAIM": "user_uuid",
}
```

- Cookies: `httpOnly=True`, `SameSite=Lax`, `secure` toggled by deployment.

### CORS

- `CORS_ALLOWED_ORIGINS` from env; **`CORS_ALLOW_CREDENTIALS = True`** is required
  because auth uses cookies.

---

## 3. App Structure

Each app follows the same file set:

```
<app>/
├── models.py        # All models for the app
├── serializers.py   # ModelSerializers (output) (+ any shared input serializers)
├── views.py         # APIView classes
├── urls.py          # App-local urlpatterns
├── admin.py         # ModelAdmin registration
├── signals.py       # Model signal handlers (imported in apps.py ready())
├── services/        # Business logic (only for apps with non-trivial logic)
└── migrations/
```

- `signals.py` is imported inside `AppConfig.ready()`:
  ```python
  def ready(self):
      import <app>.signals  # noqa
  ```
- Apps are registered in `INSTALLED_APPS` in `settings.py`.

---

## 4. Models

### 4.1 ID Strategy — internal PK + external UUID

This is the single most important model convention in the project.

- **Internal primary key:** the implicit Django `id` (`BigAutoField`). It is the
  real PK used for all relations, joins, and DB indexing. **Never exposed to the API.**
- **External identifier:** every model gets one extra UUID column named
  `<model>_uuid`:

  | Model | External key |
  |---|---|
  | `User` | `user_uuid` |
  | `Artist` | `artist_uuid` |
  | `Album` | `album_uuid` |
  | `Song` | `song_uuid` |
  | `Playlist` | `playlist_uuid` |
  | `PlaylistSong` | `playlist_song_uuid` |
  | `BaseShared` → `SharedSong` / `SharedPlaylist` | `shared_uuid` |

  ```python
  song_uuid = models.UUIDField(default=uuid.uuid4, unique=True)
  ```

  - `default=uuid.uuid4` (server-side, random).
  - `unique=True` (single UUID column, which auto-creates its own unique index —
    do **not** add a separate index for it).
  - `editable=False` on the User through-table only; other models omit it but keep
    the field in `read_only_fields` of serializers.

- **Why:** external clients never see or pass integer PKs; UUIDs hide record
  counts/ordering and prevent enumeration; internal FK joins stay fast with bigint.

### 4.2 Naming conventions

- **UUID field:** `<snake_case_model_name>_uuid` (e.g. `song_uuid`, `playlist_song_uuid`).
- **Boolean flags:** `is_*` / `has_*` · `is_public`, `is_uploaded_to_cloud`,
  `is_upload_complete`, `is_added` (annotated `isAdded` in API output).
- **Timestamps:** `created_at = models.DateTimeField(auto_now_add=True)`.
- **Ownership FKs:** context-dependent verb/noun — `created_by`, `owner`,
  `uploaded_by`, `shared_by` — always a `ForeignKey` to `User`.
- **Foreign keys:** lowercase model name (`artist = models.ForeignKey(Artist, ...)`,
  `album = models.ForeignKey(Album, ...)`).
- **Related names:** use `related_name` only when the default `<model>_set` is
  semantically wrong or awkward; a camelCase name (`related_name="uploadedSongs"`,
  `related_name="shared_links"`) is used when the name is consumed by the API.
- **Table / column content is `snake_case`; API output fields may be camelCase**
  (e.g. `isAdded` via serializer `BooleanField` + annotation).

### 4.3 Field type conventions

| Use case | Field |
|---|---|
| durations / counts | `PositiveIntegerField` |
| file size | `PositiveBigIntegerField` |
| MIME / short codes | `CharField(max_length=50)` |
| names / titles | `CharField(max_length=255)` |
| media uploads | `FileField` / `ImageField` with `upload_to="<dir>/"` |
| optional media | `null=True, blank=True` |
| optional relation | FK with `on_delete=models.SET_NULL, null=True, blank=True` |
| required ownership relation | FK with `on_delete=models.CASCADE` |

### 4.4 Meta conventions — ordering, indexes, constraints

Every model declares `ordering` so lists default to a deterministic order:

```python
class Meta:
    ordering = ["name"]          # alphabetical
    ordering = ["-created_at"]   # recent-first
    ordering = ["title"]
    ordering = ["order"]         # explicit user-controlled ordering
```

**Indexes** are added for the exact query patterns used in views:

```python
indexes = [
    models.Index(fields=["uploaded_by", "is_uploaded_to_cloud", "is_upload_complete"]),
    models.Index(fields=["playlist", "order"]),
]
```

- Index **FK + filter flags** that always appear together in `filter()` calls.
- Do not index the unique UUID (unique already creates an index).

**Uniqueness / constraints:**

```python
# Pair uniqueness on a join model
class Meta:
    unique_together = ("playlist", "song")

# Business-level uniqueness with a named constraint
constraints = [
    models.UniqueConstraint(fields=["song"], name="unique_shared_song"),
]
```

### 4.5 Abstract base models for shared columns

When several models share a field set + helper, use an abstract base:

```python
class BaseShared(models.Model):
    shared_uuid = models.UUIDField(default=uuid.uuid4, unique=True)
    shared_by = models.ForeignKey(User, on_delete=models.CASCADE)
    shared_at = models.DateTimeField(auto_now_add=True)
    expire_at = models.DateTimeField(null=True, blank=True, editable=True)

    class Meta:
        abstract = True
        ordering = ["-shared_at"]

    def isExpired(self):
        return bool(self.expire_at) and self.expire_at < timezone.now()
```

- Abstract base → children: `class SharedSong(BaseShared): song = models.ForeignKey(...)`.
- Base can also carry small helper methods like `isExpired()`.

### 4.6 Model helper methods

Small, read-only domain helpers are methods on the model (`isExpired()`).
Anything heavier goes into the services layer.

### 4.7 Custom User

```python
class User(AbstractUser):
    user_uuid = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)
    email = models.EmailField(unique=True)
    username = models.CharField(max_length=50)

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = ["username"]
```

- Login identifier is **email**; `username` is display-only (not unique).
- `email` is `unique=True`; `username` is not.
- Meta indexes the login field: `models.Index(fields=["email"])`.

---

## 5. Services Layer

Views must stay thin. Complex/multi-step flows live in `<app>/services/` as
**plain module functions** (not classes).

```
music/services/
├── upload_service.py      # multi-step upload pipeline
├── storage_service.py     # temp file, move, delete helpers
├── metadata_service.py    # mutagen/ffmpeg tag extraction & stripping
├── thumbnail_service.py   # PIL thumbnail generation
└── s3_service.py          # presigned URL generation
```

Rules:

- Functions take concrete inputs (`upload_song(file, user)`) and return objects
  or raise.
- Services use low-level django primitives: `default_storage`, `transaction.atomic()`,
  `get_or_create`, `ContentFile`, etc.
- Long pipelines are written as numbered, commented steps.
- On failure, resources are cleaned up (`try/except` with file deletion + re-raise).

---

## 6. Serializers

There are exactly two kinds of serializers, used for two distinct jobs.

### 6.1 `ModelSerializer` — OUTPUT (serializing DB records)

Named `<Entity>ModelSerializer`: `SongModelSerializer`, `ArtistModelSerializer`,
`AlbumModelSerializer`, `PlaylistModelSerializer`, `PlaylistSongModelSerializer`,
`SharedSongModelSerializer`, `UserModelSerializer`.

Used to render model instances in responses:

```python
data=SongModelSerializer(song, context={"request": self.request}).data
```

Conventions:

- Explicit `fields` list (never `__all__`).
- `read_only_fields` includes the `*_uuid`, ownership FK, and server-set timestamps.
- `write_only` fields via `extra_kwargs` (e.g. `password`).
- **Custom `create()`** is overridden only when parent behavior isn't enough.
- **`to_representation()`** is overridden to build absolute file URLs:
  ```python
  def to_representation(self, instance):
      representation = super().to_representation(instance)
      request = self.context.get("request")
      if request and instance.thumbnail:
          representation["thumbnail"] = request.build_absolute_uri(instance.thumbnail.url)
      return representation
  ```
- Computed/derived fields via `SerializerMethodField`:
  ```python
  artist_name = serializers.CharField(source="artist.name")
  songs = serializers.SerializerMethodField()
  def get_songs(self, obj): ...
  ```
- Related objects use **nested read-only serializers**:
  ```python
  song = SongModelSerializer(read_only=True)
  ```

### 6.2 `serializers.Serializer` — INPUT (validating payloads)

Plain serializers are used **only for input validation** — never for output.
They are declared as **nested classes inside the view** they belong to.

Naming = `<Action><Source>Serializer`, where `<Source>` is one of:

| Suffix | Data source |
|---|---|
| `Kwargs` | URL path params (`self.kwargs`) |
| `Query` | query string (`self.request.query_params`) |
| `Post` | body of a POST |
| `Patch` | body of a PATCH |
| `Delete` | body of a DELETE |

Examples from the codebase:

```python
class SongView(APIView):
    class SongKwargsSerializer(serializers.Serializer):
        song_uuid = serializers.UUIDField(required=False, allow_null=False)

    class SongQuerySerializer(serializers.Serializer):
        q = serializers.CharField(required=False, allow_blank=False)
        artist_uuid = serializers.UUIDField(required=False, allow_null=False)
        album_uuid = serializers.UUIDField(required=False, allow_null=False)

    def get(self, *args, **kwargs):
        kwargs_serializer = self.SongKwargsSerializer(data=self.kwargs)
        kwargs_serializer.is_valid(raise_exception=True)
        song_uuid = kwargs_serializer.validated_data.get("song_uuid")
```

Pattern:

1. Instantiate with `data=<source>`.
2. `serializer.is_valid(raise_exception=True)` — invalid input raises
   `ValidationError` (HTTP 400).
3. Read `serializer.validated_data.get("field")`.
4. **Never** `.save()` a plain Serializer; they are validation-only.

---

## 7. Views (APIView pattern)

All views extend `rest_framework.views.APIView` — **no ViewSets, no routers**.

### 7.1 Class-level auth configuration

```python
class SongView(APIView):
    permission_classes = [IsAuthenticated]
    authentication_classes = [CookieJWTAuthentication]
```

- Private endpoints: `[IsAuthenticated]` + `[CookieJWTAuthentication]`.
- Public endpoints:
  ```python
  permission_classes = [AllowAny]        # or []
  authentication_classes = []
  ```

### 7.2 HTTP method ↔ Python method

| HTTP | Python | Purpose |
|---|---|---|
| GET | `def get` | fetch one (by uuid) or a paginated list |
| POST | `def post` | create a resource |
| PATCH | `def patch` | partial update |
| DELETE | `def delete` | delete a resource |

### 7.3 Method flow (standard template)

**GET — one object:**

```python
def get(self, *args, **kwargs):
    user_obj = self.request.user
    kwargs_serializer = self.SongKwargsSerializer(data=self.kwargs)
    kwargs_serializer.is_valid(raise_exception=True)
    song_uuid = kwargs_serializer.validated_data.get("song_uuid")

    if song_uuid:                                   # detail
        song = get_object_or_404(Song, uploaded_by=user_obj, song_uuid=song_uuid)
        return formatted_response(
            data=SongModelSerializer(song, context={"request": self.request}).data,
            status=status.HTTP_200_OK,
        )
    # else: list
```

**GET — list (paginated):**

```python
    query_serializer = self.SongQuerySerializer(data=self.request.query_params)
    query_serializer.is_valid(raise_exception=True)

    song_objs = Song.objects.filter(
        uploaded_by=user_obj,
        is_uploaded_to_cloud=True,
        is_upload_complete=True,
    )
    q = query_serializer.validated_data.get("q")
    if q:
        song_objs = song_objs.filter(title__icontains=q)

    return paginated_response(
        queryset=song_objs,
        request=self.request,
        serializer_class=SongModelSerializer,
        context={"request": self.request},
    )
```

**POST / PATCH — create / update:**

```python
def post(self, *args, **kwargs):
    post_serializer = self.PlaylistPostSerializer(data=self.request.data)
    post_serializer.is_valid(raise_exception=True)
    playlist = Playlist.objects.create(
        owner=self.request.user, name=post_serializer.validated_data.get("name")
    )
    return formatted_response(
        data=PlaylistModelSerializer(playlist, context={"request": self.request}).data,
        message="Playlist created successfully",
        status=status.HTTP_201_CREATED,
    )
```

**DELETE:**

```python
def delete(self, *args, **kwargs):
    kwargs_serializer = self.SongKwargsSerializer(data=self.kwargs)
    kwargs_serializer.is_valid(raise_exception=True)
    song = get_object_or_404(Song, uploaded_by=self.request.user, song_uuid=...)
    song.delete()
    return formatted_response(message="Song deleted successfully", status=status.HTTP_200_OK)
```

### 7.4 Key conventions

- **Ownership scoping on every query**: `filter(owner=user_obj)`,
  `filter(uploaded_by=user_obj)`, `filter(created_by=user_obj)`,
  `filter(shared_by=self.request.user)`. A user can only read/write their own rows.
- **`get_object_or_404`** for detail lookups (scoped filter + uuid).
- **Validation of all three input classes** — `self.kwargs`, `self.request.query_params`,
  `self.request.data` — each through a dedicated nested serializer.
- **Errors returned as responses, not exceptions**: instead of raising, views return
  `formatted_response(message={"error": ...}, status=...)` for business-rule failures
  (duplicate, already exists, expired, invalid credentials, etc.).
- `context={"request": self.request}` is always passed so serializers can build
  absolute URLs.
- **Filtering** with `icontains` for search (`q`), FK-uuid filters via
  `artist__artist_uuid=...`.
- **Annotations** for computed flags use `Exists` + `OuterRef`:
  ```python
  playlist_song_subquery = PlaylistSong.objects.filter(
      playlist=OuterRef("pk"), song__song_uuid=song_uuid)
  playlist_objs = Playlist.objects.filter(owner=user_obj).annotate(
      isAdded=Exists(playlist_song_subquery))
  ```
- Order querysets explicitly (`order_by("-created_at")`, `order_by("added_at")`)
  so pagination is stable.

---

## 8. Authentication

### 8.1 Flow

1. `LoginView` / `RegisterView` (public, no auth) validate credentials, call
   `generate_jwt_for_user(user)`, then **set the tokens as HTTP-only cookies**:
   ```python
   response.set_cookie(
       key="access",
       value=token_and_user["tokens"]["access"],
       httponly=True, secure=False, samesite="Lax", path="/",
       max_age=86400,
   )
   ```
2. All subsequent requests carry the `access` cookie.
3. `CookieJWTAuthentication` (extends SimpleJWT `JWTAuthentication`):
   - reads `access` from `request.COOKIES`,
   - injects it into `request.META["HTTP_AUTHORIZATION"] = f"Bearer {access}"`,
   - delegates to parent validation.
4. Refresh via SimpleJWT `TokenRefreshView` mounted at `api/account/token/refresh/`;
   tokens are rotated client-side on 401.

### 8.2 Token utility (`account/jwt_utils.py`)

```python
class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user_obj):
        token = super().get_token(user_obj)
        token["user"] = {"username": user_obj.username, "email": user_obj.email}
        return token

def generate_jwt_for_user(user):
    token = MyTokenObtainPairSerializer.get_token(user)
    return {"tokens": {"refresh": str(token), "access": str(token.access_token)}}
```

### 8.3 Password handling

- `user_obj.check_password(...)` to verify; `set_password()` + `save()` to change.
- Password field is `write_only` in serializers; never serialized back.

---

## 9. Response Structure & Formatting

All JSON responses go through helpers in `utils/response_wrapper.py` to keep a
consistent contract for the frontend.

### 9.1 Standard response (`formatted_response`)

```python
def formatted_response(data=None, message=None, status=200):
    response_data = {
        "data": data if data is not None else [],
        "message": message,
        "status": status,
    }
    return Response(response_data, status=status)
```

Shape:

```json
{
  "data": { ... } | [ ... ] | [],
  "message": "human string" | {"error": "..."} | {"success": "..."} | null,
  "status": 200
}
```

Rules:

- `data` defaults to `[]` (never `null`) when empty.
- **Success** `message` is a plain string.
- **Error** `message` is a dict with a reason key, e.g. `{"error": "Invalid password"}`.
- `status` echoes the HTTP status code.
- Used with HTTP verb + semantic status: `201` on create, `200` on fetch/update/delete,
  `400`/`404` on errors.

### 9.2 Paginated response (`paginated_response`)

```python
class StandardPagination(PageNumberPagination):
    page_size_query_param = "page_size"
    max_page_size = 100

def paginated_response(queryset, request, serializer_class, *, context=None,
                       page_size=10, status_code=200):
    paginator = StandardPagination()
    paginator.page_size = page_size
    page = paginator.paginate_queryset(queryset, request)
    if page is not None:
        serializer = serializer_class(page, many=True, context=context or {})
        return paginator.get_paginated_response(serializer.data)
    serializer = serializer_class(queryset, many=True, context=context or {})
    return formatted_response(data=serializer.data, status=status_code)
```

Shape (DRF `PageNumberPagination` default, matches frontend `PaginatedResponse`):

```json
{ "count": 3, "next": "...?page=2", "previous": null, "results": [ ... ] }
```

- Handles `?page=` and `?page_size=` query params; `PAGE_SIZE = 10` global default.
- **All list endpoints must use `paginated_response`, not `formatted_response`.**

### 9.3 Specialized responses

Streaming endpoints return a dedicated shape (not the wrapper):
`{"url": ..., "type": ..., "song": {...}}` (JSON with a presigned URL).

---

## 10. URL Design

### 10.1 Mounting

App URLconfs are included from `project/urls.py`:

```python
urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/account/", include("account.urls")),   # dedicated prefix for auth app
    path("api/", include("music.urls")),             # feature apps live directly under /api/
]
```

- The auth app gets its own prefix (`/api/account/...`); feature apps sit flat
  under `/api/`.
- Media is served from object storage; web URLs resolve through S3 presigned URLs.

### 10.2 Route patterns

- **Trailing slashes** everywhere.
- **Resource path segments are always plural** (`songs/`, `artists/`, `playlists/`);
  action verbs stay singular (`upload/`, `stream/`, `share/`, `delete/`).
- **Dynamic segments use the UUID converter and the exact serializer field name:**
  ```python
  path("songs/<uuid:song_uuid>/", SongView.as_view())
  path("artists/<uuid:artist_uuid>/", ArtistView.as_view())
  path("playlists/<uuid:playlist_uuid>/", PlaylistView.as_view())
  ```
  Because the segment name (`song_uuid`) matches `SongKwargsSerializer` fields,
  `self.kwargs` can be passed straight into `data=self.kwargs`.
- Explicit names for account routes (`name="login_view"`) so `reverse()` works;
  feature routes are left unnamed.

### 10.3 "One class, multiple URLs" resource pattern

A single view class is mapped to several URL patterns that represent the same
resource's different actions. The view disambiguates by (a) HTTP method and
(b) presence/absence of the uuid kwarg:

```python
path("songs/", SongView.as_view()),                       # GET list / POST upload
path("songs/upload/", SongView.as_view()),                # POST upload
path("songs/<uuid:song_uuid>/", SongView.as_view()),      # GET one
path("songs/delete/<uuid:song_uuid>/", SongView.as_view()), # DELETE
```

### 10.4 Verb-style action URLs

Non-CRUD actions get explicit verb URLs instead of nested REST resources:

```
songs/upload/
songs/stream/<uuid:song_uuid>/
songs/share/
songs/share/<uuid:shared_uuid>/
songs/share/stream/<uuid:shared_uuid>/
playlists/<uuid:playlist_uuid>/songs/
playlists/songs/add/<uuid:playlist_uuid>/
playlists/songs/remove/<uuid:playlist_uuid>/
playlists/songs/<uuid:song_uuid>/      # playlists for a song (isAdded)
songs/ · artists/ · albums/ · playlists/   # list endpoints
playback-queue/
```

Summary of URL conventions:

| Convention | Example |
|---|---|
| List endpoint (plural) | `songs/`, `artists/`, `albums/`, `playlists/` |
| Detail endpoint (plural + uuid) | `songs/<uuid:song_uuid>/` |
| Action endpoint (verb) | `songs/upload/`, `songs/delete/`, `songs/stream/` |
| Sub-resource | `playlists/<uuid>/songs/` |
| Composite lookup | `playlists/songs/<uuid:song_uuid>/` |

---

## 11. Signals

Django model signals handle **side-effects of deletion** (file cleanup).

```python
@receiver(post_delete, sender=Song)
def delete_song_files(sender, instance, **kwargs):
    if instance.file:
        delete_file(instance.file)
    if instance.thumbnail:
        delete_file(instance.thumbnail)
    if instance.album and not Song.objects.filter(album=instance.album).exists():
        instance.album.delete()      # orphan cleanup
    if instance.artist and not Song.objects.filter(artist=instance.artist).exists():
        instance.artist.delete()
```

- Imported in `apps.py -> ready()`.
- `delete_file` in `storage_service.py` guards against `None` and missing objects.

---

## 12. Admin

Standard `ModelAdmin` classes registered per app:

- `list_display` includes the `*_uuid` column (readable in list view).
- `search_fields` on text columns including related via `artist__name`.
- `list_filter` on FK + boolean + datetime columns.
- `readonly_fields` for uuid, size/mime (system-set), and ownership FKs.

```python
class SongAdmin(admin.ModelAdmin):
    list_display = ("title", "artist", "album", "duration", "song_uuid", "uploaded_by")
    search_fields = ("title", "artist__name", "album__title")
    list_filter = ("artist", "album", "uploaded_by", "mime_type")
    readonly_fields = ("song_uuid", "size", "mime_type", "uploaded_by")
```

---

## 13. File Storage Abstraction

Object storage is the **single backend**; no local file fallback.

- Configured with **django-storages** via the standard Django storage setting
  (`STORAGES["default"]` / `DEFAULT_FILE_STORAGE`) pointing at the S3-compatible
  backend (`storages.backends.s3boto3.S3Boto3Storage`).
- A custom `PublicS3Boto3Storage` (`utils/storage.py`) wrapping `storages`:
  - rewrites internal MinIO endpoint URLs to public ones,
  - strips query strings from public thumbnails.
- Business code (models, views) only uses `upload_to` + `default_storage` /
  `FileField` — all through the standard Django storage API; it never branches on
  backend type.
- S3 media is served to clients via **presigned URLs** (public endpoint).

---

## 14. Testing

- Tests live in each app's `tests.py` (`django.test.TestCase` style).
- Management/utility scripts exist as custom management commands
  (`create_superuser`) and standalone scripts (`wait_for_db.py`).

---

## 15. Deployment (Docker / Gunicorn)

`BACKEND/entrypoint.sh` is the canonical boot sequence:

```sh
python wait_for_db.py                                   # block until DB is up
python manage.py migrate --noinput
python manage.py create_superuser                       # idempotent, from env
python manage.py collectstatic --noinput
exec python -m gunicorn project.wsgi:application --bind 0.0.0.0:8000 --workers 1
```

- Docker Compose services: `db` (Postgres), `minio` (S3-compatible object
  storage), `backend` (gunicorn), `frontend`, `nginx` (reverse proxy).
- Nginx owns traffic routing, frontend static serving, and `keepalive`.

---

## 16. New App / Endpoint Checklist

Use this checklist to build a new module in the same style as this project.

**Models**
- [ ] Internal PK: leave implicit `BigAutoField id`.
- [ ] External ID: `<model>_uuid = models.UUIDField(default=uuid.uuid4, unique=True)`.
- [ ] `created_at = models.DateTimeField(auto_now_add=True)`.
- [ ] Ownership FK to `User` (`created_by`/`owner`/`uploaded_by`) with CASCADE.
- [ ] Boolean flags prefixed `is_` with sensible defaults.
- [ ] `Meta.ordering` always set.
- [ ] `Meta.indexes` mirror real `filter()` patterns; no extra index on the uuid.
- [ ] `UniqueConstraint` / `unique_together` for business uniqueness.
- [ ] `on_delete`: CASCADE for owned/required, SET_NULL for optional.
- [ ] Shared column sets promoted to an abstract base model.

**Services** (if logic is non-trivial)
- [ ] `<app>/services/<domain>_service.py`, plain functions.

**Serializers**
- [ ] Output → `<Entity>ModelSerializer` in `serializers.py`.
  - explicit `fields`, `read_only_fields`, `context={"request": ...}`,
    `to_representation` for absolute file URLs.
- [ ] Input validation → nested `serializers.Serializer` in the view,
  named `<Action><Source>Serializer` (Kwargs/Query/Post/Patch/Delete).
- [ ] Input serializers are never used for output.

**Views**
- [ ] `APIView`, not ViewSet.
- [ ] `permission_classes` / `authentication_classes` at class level.
- [ ] `get/post/patch/delete` methods, one per HTTP verb.
- [ ] Validate `self.kwargs`, `query_params`, `body` each with its own serializer.
- [ ] Ownership-scope every queryset.
- [ ] Single-object GET → `get_object_or_404` + `formatted_response`.
- [ ] List GET → `paginated_response` + `icontains` search + `order_by`.
- [ ] Errors → `formatted_response(message={"error": ...}, status=...)`,
      not exceptions.
- [ ] Pass `context={"request": self.request}` to every model serializer.

**URLs**
- [ ] App-local `urls.py`, included under `/api/<prefix>/`.
- [ ] `<uuid:<field_name>>` converters matching serializer kwarg names.
- [ ] One view class mapped to multiple action URLs.

**Responses**
- [ ] `formatted_response` for single/object results & errors.
- [ ] `paginated_response` for every list.
- [ ] Consistent `{data, message, status}` shape everywhere.

**Auth**
- [ ] JWT in **HTTP-only cookies** via `CookieJWTAuthentication`.
- [ ] `AllowAny` + empty auth classes for public endpoints.
- [ ] `CORS_ALLOW_CREDENTIALS=True`.

**Side effects**
- [ ] `post_delete` signals for file cleanup + orphan cleanup, wired in `ready()`.

**Admin**
- [ ] `ModelAdmin` with `list_display`, `search_fields`, `list_filter`,
      `readonly_fields`.

---

## Appendix — Quick reference snippets

### Standard detail view

```python
class ItemView(APIView):
    permission_classes = [IsAuthenticated]
    authentication_classes = [CookieJWTAuthentication]

    class ItemKwargsSerializer(serializers.Serializer):
        item_uuid = serializers.UUIDField(required=False, allow_null=False)

    class ItemQuerySerializer(serializers.Serializer):
        q = serializers.CharField(required=False, allow_blank=False)

    def get(self, *args, **kwargs):
        user_obj = self.request.user
        kws = self.ItemKwargsSerializer(data=self.kwargs)
        kws.is_valid(raise_exception=True)
        item_uuid = kws.validated_data.get("item_uuid")

        if item_uuid:
            item = get_object_or_404(Item, owner=user_obj, item_uuid=item_uuid)
            return formatted_response(
                data=ItemModelSerializer(item, context={"request": self.request}).data,
                status=status.HTTP_200_OK,
            )

        qs = Item.objects.filter(owner=user_obj)
        qry = self.ItemQuerySerializer(data=self.request.query_params)
        qry.is_valid(raise_exception=True)
        if qry.validated_data.get("q"):
            qs = qs.filter(name__icontains=qry.validated_data["q"])
        return paginated_response(
            queryset=qs, request=self.request,
            serializer_class=ItemModelSerializer,
            context={"request": self.request},
        )
```

### Standard create view

```python
    class ItemPostSerializer(serializers.Serializer):
        name = serializers.CharField(required=True, allow_null=False)

    def post(self, *args, **kwargs):
        body = self.ItemPostSerializer(data=self.request.data)
        body.is_valid(raise_exception=True)
        item = Item.objects.create(owner=self.request.user, **body.validated_data)
        return formatted_response(
            data=ItemModelSerializer(item, context={"request": self.request}).data,
            message="Item created successfully",
            status=status.HTTP_201_CREATED,
        )
```
