# Food App

[![Python tests](https://github.com/laura-baraldi/food_kivy/actions/workflows/tests.yml/badge.svg)](https://github.com/laura-baraldi/food_kivy/actions/workflows/tests.yml)

A responsive food suggestion application built with **Python and Kivy** for desktop and Android.

Food App helps users decide what to eat by selecting a random item from their food lists. Users can create personal lists or share lists with other registered users. Food data is stored remotely in Supabase and protected with PostgreSQL Row Level Security (RLS).

The application supports persistent login. Refresh tokens are stored using the operating system's secure credential storage on desktop and Android Keystore on Android. The interface is available in Italian and English; user-entered food and list names are never translated automatically.

---

## Screenshots

The screenshots below show the English interface. User-entered food and list names remain in their original language.

|                                   Login                                   |                                       Random meal selection                                        |
| :-----------------------------------------------------------------------: | :------------------------------------------------------------------------------------------------: |
| <img src="screenshots/login.jpg" alt="Food App login screen" width="280"> | <img src="screenshots/home.jpg" alt="Selected dish with the recipe generation button" width="280"> |

|                                                 Generated recipe                                                  |                                Personal and shared lists                                |
| :---------------------------------------------------------------------------------------------------------------: | :-------------------------------------------------------------------------------------: |
| <img src="screenshots/recipe.jpg" alt="Generated recipe with servings, ingredients and instructions" width="280"> | <img src="screenshots/food-lists.jpg" alt="Personal and shared food lists" width="280"> |

|                              Food management                              |                             Navigation and language selection                              |
| :-----------------------------------------------------------------------: | :----------------------------------------------------------------------------------------: |
| <img src="screenshots/foods.jpg" alt="Food management popup" width="280"> | <img src="screenshots/menu.jpg" alt="Navigation menu with language selection" width="280"> |

## Why I Built This

This project started from a common problem: deciding what to eat. Instead of spending time trying to answer "What should we have for dinner?", I built an application that selects a meal from a list of foods we actually eat.

What began as a local Python project gradually evolved into a cross-platform application involving a responsive graphical interface, remote persistence, REST API communication, authentication, personal and shared data, PostgreSQL security policies, secure credential storage, Android packaging, and automated testing.

## Features

### Authentication and sessions

- User registration with email and password, email confirmation, login, logout, and password reset.
- Authenticated API requests using access tokens.
- Persistent login after closing and reopening the application.
- Refresh-token rotation and automatic access-token renewal when a protected request needs a fresh token.
- Secure refresh-token storage on desktop and Android.
- Local session cleanup when a refresh token is rejected.

### Personal and shared food lists

- Default list containing 22 foods for every new user.
- Multiple personal lists: create, select, rename, and delete.
- Shared lists for registered users, with member management by email.
- Owner and member roles; members can leave a shared list.
- Only the owner can add or remove other members and delete the shared list.
- Shared changes are immediately visible to authorized members.

### Food management and interface

- Add and remove foods, with case-insensitive duplicate detection.
- Random food selection from the active list.
- Italian and English interface text, with language preference saved locally.
- User-entered list names and food names remain in their original language.
- Responsive Kivy layouts, reusable rounded buttons, custom hamburger menu, scrollable screens and popups.
- Density-independent dimensions with `dp` and scalable text with `sp`.
- Adaptive backgrounds for different screen aspect ratios.
- Android software-keyboard handling.
- Custom fonts, illustrations, flag images, and application icon.
- Desktop and Android support.

### Testing and automation

- Automated tests for application behaviour, authentication, storage, screens, popups, shared-list permissions, and both interface languages.
- 100% statement-coverage requirement for the Python modules measured in CI.
- Tests for secure-storage integration without accessing real credentials.
- GitHub Actions CI on pushes and pull requests.
- Headless Kivy testing with Xvfb.
- Android APK build workflow.
- Versioned PostgreSQL migrations.

## How It Works

At startup, the application loads the saved language preference and checks for a refresh token in the device's secure credential storage.

- If a token is available, the application asks Supabase to renew the session.
- If no valid session is available, the login screen is displayed.
- After authentication, the application loads the personal and shared lists available to the user.
- The user can create or select a list, add or remove foods, and request a random selection.

After login, Supabase returns an access token and a refresh token. The access token is used for authenticated requests; the refresh token is saved in secure credential storage so the session can be restored after a restart. When Supabase renews a session, the application replaces the previous refresh token with the new one. Renewal happens when the application reopens or when a protected request needs a fresh access token; it does not run continuously in the background.

If at least one list exists, the first available list becomes active automatically. Changes are sent immediately to Supabase, so lists and foods remain available after the application is closed. Logging out clears the saved refresh token and application state, then requests server-side sign-out when Supabase is reachable.

Authentication redirects are handled by the GitHub Pages site:

```text
https://laura-baraldi.github.io/food_kivy/
```

## Application Structure

The code is organized by responsibility. `main.py` coordinates application state, session renewal, the active list, language changes, and screen navigation.

```text
food_kivy/
├── .github/
│   └── workflows/
│       ├── build-apk.yml
│       └── tests.yml
├── app/
│   ├── auth/
│   │   ├── auth_client.py
│   │   ├── auth_screen.py
│   │   └── signup_screen.py
│   ├── core/
│   │   ├── android_token_store.py
│   │   ├── desktop_token_store.py
│   │   ├── session_storage.py
│   │   └── supabase_client.py
│   ├── food/
│   │   ├── food_lists_screen.py
│   │   ├── food_list_popups.py
│   │   ├── food_popup.py
│   │   └── group_members_popup.py
│   ├── i18n/
│   │   ├── language_settings.py
│   │   └── translations.py
│   ├── recipes/
│   │   ├── recipe_client.py
│   │   └── recipe_popup.py
│   └── ui/
│       └── ui_components.py
├── android_src/
│   └── org/foodkivy/security/TokenVault.java
├── docs/
│   └── index.html
├── fonts/
├── images/
├── screenshots/
├── supabase/
│   └── migrations/
├── tests/
├── main.py
├── make_flags.py
├── buildozer.spec
├── pytest.ini
└── README.md
```

The `app` package separates authentication, platform-specific token storage and shared services, food-list features, recipes, translations, and reusable UI components.

The application uses standard HTTP requests rather than the full Supabase Python SDK. This keeps the Android dependency tree smaller and avoids transitive packages that are difficult to cross-compile with python-for-android.

## Persistent Login and Token Storage

`app/core/session_storage.py` provides common save, load, and delete operations while selecting the storage implementation for the current platform.

- **Desktop:** `app/core/desktop_token_store.py` uses `keyring` to access a supported operating-system credential store, such as Windows Credential Manager, macOS Keychain, or a supported Linux Secret Service/KWallet backend.
- **Android:** `app/core/android_token_store.py` uses PyJNIus to call `TokenVault.java`. The Java class uses Android Keystore for the encryption key and stores the encrypted refresh token in application-private preferences.

If secure storage is unavailable, the application does not save the refresh token in plaintext. The user may need to sign in again after restarting.

Older versions stored the refresh token in a local `session.json` file. When that file is encountered, the application attempts to move its token to secure storage and removes the old plaintext file. If migration cannot be completed securely, the old file is removed and the user must sign in again.

Tests replace operating-system storage with in-memory fakes. They do not read or modify a developer's real saved credentials.

## Language Settings

`app/i18n/translations.py` contains the Italian and English interface strings. Screens and popups request a string by its key and display it in the selected language.

`app/i18n/language_settings.py` saves the selected language in the application's local data directory and restores it at startup. The language controls appear on the login and registration screens, as well as in the home and food-list menus. A popup opened after a language change uses the selected language.

Only interface text is translated. A list named `Cena`, for example, remains `Cena` when the interface is switched to English. Food names, list names, and email addresses are treated as user data.

## Personal and Shared Lists

A personal list belongs to one user through `owner_id`. Its owner can view it, rename or delete it, and view, add, or remove its foods.

A shared list is connected to an internal group through `group_id`. Groups do not have a separate screen; users interact with personal and shared lists from the same list-management screen.

When a shared list is created:

1. The application creates a group.
2. The creator becomes its owner and is automatically added as a group member.
3. The food list is connected to the group.
4. The owner can add registered users by email.

| Action                             | Owner | Member |
| ---------------------------------- | :---: | :----: |
| View the shared list and its foods |  Yes  |  Yes   |
| Add and remove foods               |  Yes  |  Yes   |
| Rename the list                    |  Yes  |  Yes   |
| View members                       |  Yes  |  Yes   |
| Add members                        |  Yes  |   No   |
| Remove other members               |  Yes  |   No   |
| Leave the list                     |  No   |  Yes   |
| Delete the list for everyone       |  Yes  |   No   |

A regular member can leave without deleting the list for the remaining users. Deleting a shared list removes its foods, memberships, and internal group through PostgreSQL cascade rules. The owner cannot currently leave or transfer ownership.

## Supabase REST Integration

The REST endpoints are built from the Supabase project URL. Protected requests include the authenticated user's access token:

```python
FOODS_URL = f"{SUPABASE_URL}/rest/v1/foods"
headers["Authorization"] = f"Bearer {access_token}"
```

The application uses:

- `GET` to retrieve lists, foods, groups, and members;
- `POST` to create records and call PostgreSQL functions;
- `PATCH` to rename lists and groups;
- `DELETE` to remove foods, lists, groups, and memberships;
- RPC endpoints for protected membership operations.

The access token identifies the current user. Database policies determine which records that user is allowed to access.

## Database Model and Security

The main PostgreSQL tables are:

- `default_foods` — template used to populate a new user's default list;
- `food_lists` — personal and shared lists;
- `foods` — food items connected to lists through `list_id`;
- `groups` — internal ownership records for shared lists;
- `group_members` — users allowed to access shared lists.

A personal list has an `owner_id`; a shared list has a `group_id`. A database constraint ensures that a list belongs either to one user or to one group, never both. The `group_members` table stores the `owner` and `member` roles. When a group is created, a PostgreSQL trigger automatically inserts its creator as the owner.

Database functions support authorization and membership management, including checking group membership and ownership, checking access to a food list, retrieving members, and adding an existing user by email.

Row Level Security is enabled for application data. Policies enforce that:

- anonymous users cannot access food data;
- users can access only their own personal lists;
- users can access shared lists only when they belong to the corresponding group;
- group members can view and modify foods in an accessible shared list;
- only the owner can add or remove other members;
- members cannot add themselves to arbitrary groups;
- regular members can remove only their own membership when leaving;
- the owner's membership cannot be removed;
- only the owner can delete a shared list and its group.

These permissions are enforced in PostgreSQL, not only by hiding buttons in the interface. The application uses the Supabase publishable key or legacy `anon` key together with the authenticated user's access token. The `service_role` key must never be included in the client application.

The SQL schema, policies, functions, and triggers are stored as versioned migrations in `supabase/migrations/`.

## User Interface and Responsive Design

The application uses a Kivy `ScreenManager` to navigate between the login screen, registration screen, main food screen, and food-list management screen.

- **Main screen:** displays the active food list, random-choice button, selected result, and hamburger navigation button. If no list is active, the user is asked to create one; if the list is empty, the user is asked to add foods.
- **Food-list screen:** displays personal and shared lists together. Shared lists are marked as `(condivisa)` in Italian or `(shared)` in English. The available actions depend on whether the user owns or belongs to the list.
- **Food-management popup:** contains a scrollable food list, one text field, add/delete buttons, and status feedback. The same input is used for both actions.
- **Member-management popup:** displays all users who can access a shared list. All members can view participants; only the owner can add or remove other members.
- **Reusable components:** `RoundedButton` draws its background with a Kivy `RoundedRectangle`. `MenuButton` extends it and draws the hamburger icon using three canvas lines.

Desktop and Android devices have different resolutions, pixel densities, physical dimensions, and aspect ratios. The interface uses:

- `dp` for widget dimensions, spacing, padding, and rounded corners;
- `sp` for text sizes;
- `size_hint` and `pos_hint` for proportional sizing and positioning;
- scrollable layouts for content that may exceed the available space.

For example:

```python
self.choose_button.height = dp(48)
self.title_label.font_size = sp(34)
```

### Adaptive backgrounds

Background images use `fit_mode="cover"` to fill the available screen without stretching the illustration. Because tall mobile displays crop images differently, the home screen calculates the aspect ratio and selects an appropriate asset:

```python
ratio = Window.width / Window.height

background_source = (
    "images/background_tall.png"
    if ratio < 0.48
    else "images/background.png"
)
```

### Mobile keyboard handling

On Android, the software keyboard can cover form controls. The application uses:

```python
Window.softinput_mode = "below_target"
```

When the food input receives focus, the popup scrolls toward the field so that the input and buttons remain visible. The focus handler runs only on Android:

```python
if platform != "android" or not focused:
    return
```

This prevents the popup from moving unnecessarily when the field is selected on desktop.

## Project Evolution

The project developed incrementally:

1. **Local Python prototype:** stored foods in a local JSON file and implemented random selection, add/delete operations, and simple persistence.
2. **Kivy graphical interface:** introduced widgets, layouts, labels, buttons, text inputs, popups, custom fonts, and image assets.
3. **Supabase persistence:** replaced local JSON storage with PostgreSQL hosted on Supabase and introduced REST API communication.
4. **Android packaging:** added Buildozer and python-for-Android, portrait orientation, Internet permission, a custom icon, and a reduced dependency set.
5. **Dependency redesign:** replaced the full Supabase Python SDK with direct `requests` calls because some transitive dependencies were difficult to cross-compile for Android.
6. **Responsive redesign:** addressed pixel density, aspect ratios, background cropping, popup positioning, and the software keyboard using `dp`, `sp`, adaptive images, proportional positioning, and scrollable content.
7. **Authentication and security:** added Supabase Authentication, email confirmation, password reset, access tokens, protected requests, and PostgreSQL RLS.
8. **Multiple personal lists:** introduced multiple lists and automatic creation of a default list for every new user.
9. **Shared lists:** added groups, memberships, roles, protected PostgreSQL functions, database triggers, cascade deletion, and membership-based access control.
10. **Automated testing and CI:** expanded tests to HTTP clients, application state, Kivy screens, popups, reusable components, list permissions, Android-specific behaviour, and error paths.
11. **Persistent login:** added session restoration, refresh-token rotation, desktop credential stores, Android Keystore, and migration away from the older plaintext session file.
12. **Italian and English interface:** moved interface strings into a shared translation table and saved the language preference locally.

## Technologies

| Area                | Technologies                                                                |
| ------------------- | --------------------------------------------------------------------------- |
| Application         | Python 3, Kivy, Requests                                                    |
| Desktop security    | `keyring`, operating-system credential stores                               |
| Android             | PyJNIus, Java, Android Keystore, Buildozer, python-for-Android              |
| Backend and data    | Supabase, Supabase Authentication, PostgreSQL, REST API                     |
| Database security   | Row Level Security, PostgreSQL functions, triggers, constraints, migrations |
| Testing             | pytest, pytest-cov, `unittest.mock`, Xvfb                                   |
| Automation          | GitHub Actions                                                              |
| Web and development | HTML, CSS, Git, GitHub, GitHub Pages                                        |

## Configuration

Create `config.py` in the project root:

```python
SUPABASE_URL = "your-supabase-project-url"
SUPABASE_KEY = "your-supabase-publishable-or-anon-key"
```

Use the Supabase publishable key or legacy `anon` key. Never place the `service_role` key inside the application. `config.py` is included in `.gitignore` and must not be committed with real project values.

## Run Locally

Clone the repository:

```bash
git clone https://github.com/laura-baraldi/food_kivy.git
cd food_kivy
```

Optionally create a virtual environment:

```bash
python -m venv .venv
```

Activate it on Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Install the desktop dependencies:

```bash
python -m pip install kivy requests keyring pytest pytest-cov
```

Create `config.py` with the required Supabase values, then run:

```bash
python main.py
```

Persistent login requires an available, supported operating-system credential store. If secure storage is unavailable, the application can still run, but the user may need to sign in again after closing it.

## Tests and Coverage

Run the complete test suite:

```bash
python -m pytest
```

Run tests with the same coverage requirement used by GitHub Actions:

```bash
python -m pytest --cov=app.auth.auth_client --cov=app.core.android_token_store --cov=app.core.desktop_token_store --cov=app.auth.auth_screen --cov=app.food.food_lists_screen --cov=app.food.food_list_popups --cov=app.food.food_popup --cov=app.food.group_members_popup --cov=app.i18n.language_settings --cov=main --cov=app.recipes.recipe_client --cov=app.recipes.recipe_popup --cov=app.core.session_storage --cov=app.auth.signup_screen --cov=app.core.supabase_client --cov=app.i18n.translations --cov=app.ui.ui_components --cov-report=term-missing --cov-fail-under=100
```

GitHub Actions requires 100% statement coverage across the selected Python modules. Tests run automatically on every push and pull request. Xvfb provides a virtual display for Kivy tests on the GitHub Actions Ubuntu runner.

The Python tests do not compile or execute `TokenVault.java`. The Android build verifies compilation, and installation on a physical device verifies the login and logout flow.

## Android Build

Android packaging is configured in `buildozer.spec`. The application requirements include:

```ini
requirements = python3,kivy,requests,certifi,urllib3,idna,charset_normalizer
```

The Android configuration includes:

```ini
orientation = portrait
android.permissions = INTERNET
android.api = 35
android.minapi = 24
android.ndk_api = 24
android.add_src = android_src
```

`android.add_src` includes the Java source that implements secure token storage through Android Keystore. The application icon is configured with:

```ini
icon.filename = images/icon.png
```

The **Build APK** GitHub Actions workflow runs automatically on pushes to `main` and can also be started manually for a selected branch using **Run workflow**.

The Android APK has been tested on a physical phone for persistent login and logout: the application retained the login after being closed and reopened, and returned to the login screen after logout and reopening.

## Current Limitations

- Users must already have an account before they can be added to a shared list; there is no invitation flow for users without an account.
- Group ownership cannot currently be transferred.
- Database requests are synchronous.
- The application requires an Internet connection for remote data operations.
- Network errors have limited user-facing feedback.
- Offline caching and retry handling are not implemented.
- Persistent login depends on an available secure credential store on the device.
- Access tokens that have already been issued remain valid until their expiry, even after the corresponding refresh token is revoked.

## Possible Future Improvements

- Shared-list invitations and group ownership transfer.
- Improved loading indicators and network-error feedback.
- Asynchronous requests, retry handling, offline caching, and synchronization after reconnecting.
- Food categories, filters, and favourites.
- Additional languages and further interface improvements.

## What I Learned

This project provided practical experience with:

**Python and application structure**

- Separating responsibilities between modules, object-oriented programming, inheritance, and application state management.
- Reusable functions and UI components, external configuration, validation, and exception handling.
- Synchronization between local and remote state, translations, and locally saved preferences.

**Kivy and Android**

- Widgets, layouts, screen navigation, popups, dropdown menus, canvas drawing, custom fonts, and images.
- Responsive design using `dp` and `sp`, scrollable mobile layouts, and software-keyboard/focus handling.
- Buildozer, python-for-Android, Android dependency compatibility, APK creation, and physical-device testing.
- Calling Android Java APIs from Python through PyJNIus.

**APIs, databases, and security**

- REST API communication, HTTP methods, headers, and JSON payloads.
- Supabase Authentication, access and refresh tokens, session renewal, and refresh-token rotation.
- Operating-system credential storage and Android Keystore.
- PostgreSQL relational modelling, foreign keys, cascade deletion, RLS, membership-based authorization, functions, triggers, constraints, and migrations.

**Testing and Git**

- pytest, pytest-cov, mocking HTTP requests, and patching application dependencies.
- Testing Kivy widgets, nested callbacks, platform-specific behaviour, and Italian/English interface behaviour.
- Isolating tests from real credentials and maintaining a 100% statement-coverage requirement.
- Git branches, commits, pull requests, merges, GitHub Actions, and automated builds.

## Status

The current version includes registration, email confirmation, login, logout, password reset, persistent login with secure refresh-token storage, automatic session renewal, authenticated Supabase requests, multiple personal lists, default list creation with 22 foods, shared multi-user lists, owner/member permissions, member management by email, PostgreSQL RLS, persistent remote data, random food selection, Italian and English interface text, responsive desktop and Android layouts, custom application branding, and automated tests with a 100% statement-coverage requirement in CI.

The authentication flow, personal lists, shared lists, member management, and food operations work on desktop. The persistent-login and logout flow has also been verified on a physical Android phone.
