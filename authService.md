

## 🔐 1. `/auth/login` (POST)

**Purpose**: Authenticate a user and return a JWT token.

### Business Logic

```java
public LoginResponse login(LoginRequest request) {

    // 1. Find user by email
    User user = userRepository.findByEmail(request.getEmail())
        .orElseThrow(() -> new RuntimeException("Invalid email or password"));

    // 2. Verify password
    if (!passwordEncoder.matches(request.getPassword(), user.getPasswordHash())) {
        throw new RuntimeException("Invalid email or password");
    }

    // 3. Generate JWT Token
    String token = jwtService.generateToken(user);

    // 4. Update last login
    user.setLastLogin(LocalDateTime.now());
    userRepository.save(user);

    // 5. Return response
    return new LoginResponse(token, UserMapper.toDTO(user));
}
```

---

## 👥 2. `/users` (GET) — Admin

**Purpose**: Retrieve all users (admin only).

### Business Logic

```java
public List<UserDTO> getAllUsers() {

    List<User> users = userRepository.findAll();

    return users.stream()
            .map(UserMapper::toDTO)
            .toList();
}
```

---

## ➕ 3. `/users` (POST) — Create User

**Purpose**: Create a new user.

### Business Logic

```java
public UserDTO createUser(UserDTO dto) {

    // 1. Check duplicate email
    if (userRepository.existsByEmail(dto.getEmail())) {
        throw new RuntimeException("Email already exists");
    }

    // 2. Fetch role
    Role role = roleRepository.findById(dto.getRoleId())
        .orElseThrow(() -> new RuntimeException("Role not found"));

    // 3. Convert DTO → Entity
    User user = new User();
    user.setName(dto.getName());
    user.setEmail(dto.getEmail());

    // 4. Encrypt password
    user.setPasswordHash(passwordEncoder.encode(dto.getPassword()));

    user.setRole(role);
    user.setCreatedAt(LocalDateTime.now());

    // 5. Save
    userRepository.save(user);

    // 6. Return DTO
    return UserMapper.toDTO(user);
}
```

---

## ✏️ 4. `/users/{userId}` (PUT) — Update User

**Purpose**: Partially update an existing user.

### Business Logic

```java
public UserDTO updateUser(Integer userId, UserDTO dto) {

    // 1. Fetch existing user
    User user = userRepository.findById(userId)
        .orElseThrow(() -> new RuntimeException("User not found"));

    // 2. Update fields only if present
    if (dto.getName() != null) {
        user.setName(dto.getName());
    }

    if (dto.getEmail() != null) {
        user.setEmail(dto.getEmail());
    }

    if (dto.getRoleId() != null) {
        Role role = roleRepository.findById(dto.getRoleId())
            .orElseThrow(() -> new RuntimeException("Role not found"));
        user.setRole(role);
    }

    // 3. Save
    userRepository.save(user);

    return UserMapper.toDTO(user);
}
```

---

## 🏷️ 5. `/roles` (GET)

**Purpose**: Fetch all available roles.

### Business Logic

```java
public List<RoleDTO> getAllRoles() {

    List<Role> roles = roleRepository.findAll();

    return roles.stream()
            .map(RoleMapper::toDTO)
            .toList();
}
```

---

## 🔒 6. Security Rules

The following endpoints are restricted to users with the **ADMIN** role:

- `GET /users`
- `POST /users`
- `PUT /users/{userId}`
- `GET /roles`


```
