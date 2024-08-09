# User Authentication and Authorization Implementation

## Table of Contents
- [Overview](#overview)
- [User](#user)
    - [User Entity](#user-entity)
    - [User Repository](#user-repository)
    - [User Service](#user-service)
    - [User Controller](#usercontroller)
    - [User Profile](#user-profile)
    - [Role](#role)
- [Token](#token)
- [DTOs (Data Transfer Object)](#dtos-data-transfer-object)
    - [User Response](#userresponsedto)
    - [User Registration Response](#userregistrationresponsedto)
    - [User Login Response](#userloginresponsedto)
    - [User Delete Response](#userdeleteresponsedto)
    - [User Profile Response](#userprofileresponsedto)
    - [User Logout Response](#userlogoutdto)
    - [User Logout Request](#userlogoutrequestdto)
    - [Token Authentication Response](#tokenauthenticationresponsedto)
    - [Token Refresh Request](#tokenrefreshrequestdto)

- [Security](#security)
    - [Security Configuration](#security-configuration)
    - [Token Service](#token-service)
    - [JWT Auth Entry Point](#jwtauthentrypoint)
    - [JWT Authentication Filter](#jwtauthenticationfilter)
    - [Custom User Details Service](#customuserdetailsservice)
    - [Security Contants](#securityconstants)

## Overview
This document provides a detailed high-level design for the implementation of user authentication and authorization in the fitness app. The app supports two roles: User and Trainer. Users can sign up with or without a Trainer ID. The authentication process includes user registration, login, and JWT token management using Spring Boot and AWS.

---

## User

### User Entity

The `UserEntity` represents the user in the database.

### Description
- **Imports**: Required classes for JPA and persistence.
- **Annotations**:
  - `@Entity` to mark the class as a JPA entity.
  - `@Id` and `@GeneratedValue` for primary key generation.
  - `@ElementCollection` to store the roles as a list.
  - `@PrePersist` to set the creation timestamp and default role.
  - `@OneToOne` for mapping the user profile.
  - `@JsonManagedReference` for handling bidirectional relationships in JSON serialization.
  - `@Table` to specify the table name in the database.
  - `@Data` and `@NoArgsConstructor` from Lombok to generate getters, setters, and a no-argument constructor.
- **Fields**:
  - `id`: Unique identifier for the user.
  - `username`: The username chosen by the user.
  - `email`: The user's email address.
  - `password`: The user's hashed password.
  - `trainerId`: Optional identifier linking the user to a specific trainer.
  - `createdAt`: The timestamp of when the user was created.
  - `role`: List of roles assigned to the user.
  - `profile`: User's profile information, linked through a one-to-one relationship.
- **Methods**:
  - `onCreate()`: Sets the `createdAt` timestamp to the current instant, initializes the role to `Role.USER`, and creates a new `UserProfile` for the user.
  - `isAccountNonExpired()`, `isAccountNonLocked()`, `isCredentialsNonExpired()`, `isEnabled()`: Methods to indicate the account's status.
  - `toString()`: Provides a string representation of the user entity.

### PseudoCode
```java
import jakarta.persistence.*;
import java.util.*;
import lombok.*;

@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
public class UserEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;
    private String email;
    private String password;
    private String trainerId;
    private Instant createdAt;

    @ElementCollection(fetch = FetchType.EAGER)
    @Enumerated(EnumType.STRING)
    private List<Role> roles = new ArrayList<>();

    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL)
    @JsonManagedReference
    private UserProfile profile;

    @PrePersist
    protected void onCreate() {
        this.createdAt = Instant.now();
        this.roles = List.of(Role.USER);
        this.profile = new UserProfile(this);
    }

    public boolean isAccountNonExpired() {
        // Implementation
    }

    public boolean isAccountNonLocked() {
        // Implementation
    }

    public boolean isCredentialsNonExpired() {
        // Implementation
    }

    public boolean isEnabled() {
        // Implementation
    }

    @Override
    public String toString() {
        // Implementation
    }
}
```

[Back to Top](#top)

---

### User Repository

The UserRepository interface handles database operations for the User entity.

### Description
- **Imports**: Required classes for Spring Data JPA and `UserEntity` object
- **Annotations**: `@Repository` to mark the interface as a Spring Data repository.
- **Methods**:
  - `findByUsername()`: Custom query to find a user by their username.
  - `existsByUsername()`: Custom query to check if a user with the given username exists.

### PseudoCode
```java
import path.to.user.UserEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.*;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<UserEntity> findByUsername(String);

    Boolean existsByUsername(String);
}
```
[Back to Top](#top)


---

### User Service

The `UserService` handles the business logic for user registration and authentication.

### Description
- **Imports**: Required classes for DTOs, model, repository, security, logging, and Spring framework.
- **Annotations**:
  - `@Slf4j` to enable logging.
  - `@Service` to mark the class as a service.
  - `@Autowired` to inject dependencies.
- **Constructor**:
  - Autowired constructor to inject dependencies.
- **Fields**:
  - `UserRepository`: Injected to manage user persistence.
  - `PasswordEncoder`: Injected to handle password encoding.
  - `AuthenticationManager`: Injected to manage authentication.
  - `TokenService`: Injected to handle JWT token operations.
- **Methods**:
  - `registerUser()`: Handles the registration process by creating a new `UserEntity` object, hashing the password, assigning roles, and saving the user to the database.
  - `loginUser()`: Manages user authentication and JWT token generation.
  - `updateUser()`: Updates an existing user's details based on the provided user ID and user details.
  - `deleteUser()`: Deletes a user based on the provided user ID.
  - `getAllUsers()`: Retrieves a list of all users.
  - `getUserByUsername()`: Retrieves a user by their username.
  - `getUserById()`: Retrieves a user by their ID.
  - `generateJwtToken()`: Generates a JWT token for the authenticated user.
  - `generateRefreshToken()`: Generates a refresh token for the authenticated user.
  - `validateJwtToken()`: Validates the provided JWT token using the `TokenService` method.
  - `userResponseDtoWithTokens()`: Creates a `UserResponseDto` with both access and refresh tokens and user details.
  - `getUserProfile()`: Retrieves the user profile.
  - `getUsernameFromJwtToken()`: Retrieves the username from the provided JWT token.

### PseudoCode
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.*;

@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private TokenService tokenService;

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder, AuthenticationManager authenticationManager, TokenService tokenService) {
        // Constructor implementation
    }

    public UserResponseDto registerUser(UserRegistrationDto registrationDto) {
        // Implementation
    }

    @Transactional
    public UserResponseDto loginUser(UserLoginDto loginDto) {
        // Implementation
    }

    public UserEntity updateUser(Long id, UserEntity userDetails) {
        // Implementation
    }

    public UserDeleteResponseDto deleteUser(Long id) {
        // Implementation
    }

    public List<UserEntity> getAllUsers() {
        // Implementation
    }

    public UserEntity getUserByUsername(String username) {
        // Implementation
    }

    public UserEntity getUserById(Long id) {
        // Implementation
    }

    public String generateJwtToken(Authentication authentication) {
        // Implementation
    }

    public String generateRefreshToken(Authentication authentication) {
        // Implementation
    }

    public boolean validateJwtToken(String authToken) {
        // Implementation
    }

    private UserResponseDto userResponseDtoWithTokens(String accessToken, String refreshToken, UserEntity user) {
        // Implementation
    }

    public UserResponseDto userResponseDtoWithToken(String accessToken, UserEntity user) {
        // Implementation
    }

    public UserProfileResponseDto getUserProfile(Long id) {
        // Implementation
    }

    public String getUsernameFromJwtToken(String token) {
        // Implementation
    }
}
```

[Back to Top](#top)
---

### UserController

The `UserController` exposes endpoints for user registration and authentication.

### Description
- **Imports**: Required classes for DTOs, model, service, Spring MVC, and logging.
- **Annotations**:
  - `@Slf4j` to enable logging.
  - `@RestController` to mark the class as a REST controller.
  - `@RequestMapping` to define the base URL for the endpoints.
  - `@RequestBody` to bind the incoming HTTP request body to the method parameter.
  - `@PostMapping` to map HTTP POST requests to the specific handler methods.
- **Constructor**:
  - Autowired constructor to inject dependencies.
- **Fields**:
  - `UserService`: Injected to handle user operations.
- **Methods**:
  - `registerUser()`: Handles user registration requests by calling the `registerUser` method in `UserService` and returning the created user.
  - `loginUser()`: Handles user login requests by calling the `loginUser` method in `UserService` and returning a JWT token and user details.
  - `updateUser()`: Updates a user's details.
  - `deleteUser()`: Handles the deletion of a user.
  - `getAllUsers()`: Retrieves all users.
  - `getUserById()`: Retrieves a user by their ID.
  - `logoutUser()`: Logs out a user by validating the JWT token and deleting it from the system.
  - `getUserProfile()`: Retrieves the profile of a user by their ID, validating the JWT token for access.
  - `refreshToken()`: Refreshes the JWT token if the provided refresh token is valid.

### PseudoCode
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/user")
public class UserController {

    @Autowired
    private UserService userService;

    @Autowired
    private TokenService tokenService;

    @Autowired
    public UserController(UserService userService, TokenService tokenService) {
        this.userService = userService;
        this.tokenService = tokenService;
    }

    @PostMapping("auth/register")
    public ResponseEntity<UserResponseDto> registerUser(@RequestBody UserRegistrationDto userDto) {
        // Implementation
    }

    @PostMapping("auth/login")
    public ResponseEntity<UserResponseDto> loginUser(@RequestBody UserLoginDto loginDto) {
        // Implementation
    }

    @PutMapping("update/{userId}")
    public ResponseEntity<UserResponseDto> updateUser(@PathVariable Long userId, @RequestBody UserUpdateDto userUpdateDto) {
        // Implementation
    }

    @DeleteMapping("delete/{userId}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long userId) {
        // Implementation
    }

    @GetMapping("all/{userId}")
    public ResponseEntity<List<UserResponseDto>> getAllUsers() {
        // Implementation
    }

    @GetMapping("admin/auth/{userId}")
    public ResponseEntity<UserResponseDto> getUserById(@PathVariable Long userId) {
        // Implementation
    }

    @PostMapping("/logout")
    public ResponseEntity<UserLogoutDto> logoutUser(@RequestHeader("Authorization") String token, @RequestBody UserLogoutRequestDto logoutDto) {
        // Implementation
    }

    @GetMapping("profile/{userId}")
    public ResponseEntity<UserProfileResponseDto> getUserProfile(@PathVariable Long userId, @RequestHeader("Authorization") String token) {
        // Implementation
    }

    @PostMapping("/refresh")
    public ResponseEntity<?> refreshToken(HttpServletRequest request, HttpServletResponse response) {
        // Implementation
    }
}
```

[Back to Top](#top)

---

### UserProfile

The `UserProfile` entity represents the profile information of a user in the database.

### Description
- **Imports**: Required classes for JPA and persistence.
- **Annotations**:
  - `@Entity` to mark the class as a JPA entity.
  - `@Table` to specify the table name in the database.
  - `@Id` and `@GeneratedValue` for primary key generation.
  - `@ElementCollection` to store the user's goals as a list.
  - `@OneToOne` and `@JoinColumn` to map the one-to-one relationship with the `UserEntity`.
  - `@JsonBackReference` for handling bidirectional relationships in JSON serialization.
  - `@Data` and `@NoArgsConstructor` from Lombok to generate getters, setters, and a no-argument constructor.
- **Fields**:
  - `id`: Unique identifier for the user profile.
  - `firstName`: The user's first name.
  - `lastName`: The user's last name.
  - `age`: The user's age.
  - `weight`: The user's weight.
  - `height`: The user's height.
  - `goals`: A list of the user's fitness goals.
  - `user`: The associated `UserEntity` object.
- **Constructor**:
  - `UserProfile(UserEntity user)`: Initializes the user profile with default values and associates it with a `UserEntity`.

### PseudoCode
```java
import jakarta.persistence.*;
import java.util.*;
import lombok.*;

@Entity
@Table(name = "user_profile")
@Data
@NoArgsConstructor
public class UserProfile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String firstName;
    private String lastName;
    private int age;
    private float weight;
    private float height;

    @ElementCollection(fetch = FetchType.EAGER)
    private List<String> goals;

    @OneToOne
    @JoinColumn(name = "user_id")
    @JsonBackReference
    private UserEntity user;

    public UserProfile(UserEntity user) {
        this.user = user;
        this.firstName = "";
        this.lastName = "";
        this.age = 0;
        this.weight = 0.0f;
        this.height = 0.0f;
        this.goals = new ArrayList<>();
    }
}
```

[Back to Top](#top)

---

### Role

The Role enum defines different roles a user can have within the system.

### Description
- **Enum Constants**:
  - `USER`: Represents a regular user.
  - `TRAINER`: Represents a trainer.

### PseudoCode
```java
package com.evoluir.evoluirfitness.model.user;

public enum Role {
    USER,
    TRAINER
}
```

[Back to Top](#top)

---
### Token

The `Token` entity represents a refresh token in the database.

### Description
- **Imports**: Required classes for JPA and persistence.
- **Annotations**:
  - `@Entity` to mark the class as a JPA entity.
  - `@Table` to specify the table name in the database.
  - `@Id` and `@GeneratedValue` for primary key generation.
  - `@Column` to specify unique constraints.
  - `@ManyToOne` and `@JoinColumn` to map the many-to-one relationship with the `UserEntity`.
  - `@Data` from Lombok to generate getters, setters, and other utility methods.
- **Fields**:
  - `id`: Unique identifier for the token.
  - `token`: The actual refresh token string.
  - `expiryDate`: The date and time when the token expires.
  - `revoked`: Indicates if the token has been revoked.
  - `expired`: Indicates if the token has expired.
  - `username`: The username associated with the token.
  - `user`: The associated `UserEntity` object.

### PseudoCode
```java
package com.evoluir.evoluirfitness.token.model;

import com.evoluir.evoluirfitness.user.model.UserEntity;
import jakarta.persistence.*;
import lombok.Data;
import java.time.Instant;

@Entity
@Table(name = "refresh_tokens")
@Data
public class Token {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String token;

    private Instant expiryDate;
    public boolean revoked;
    private boolean expired;
    private String username;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    public UserEntity user;
}
```

[Back to Top](#top)


---

## DTOs (Data Transfer Object)
DTOs are used encapsulate data and transfer it from one part of an application to another

---

### UserResponseDto

The UserResponseDto is used to transfer user data in responses for both login and registration.


### Description
- **Imports**: Required classes for model, Lombok, and Java time.
- **Annotations**:
  - `@Data` from Lombok to generate getters and setters.
- **Fields**:
  - `id`: The unique identifier for the user.
  - `username`: The username of the user.
  - `email`: The email address of the user.
  - `role`: The list of roles assigned to the user.
  - `createdAt`: The timestamp of when the user was created.
  - `token`: The JWT token generated upon login.
  - `tokenType`: The type of the token, typically "Bearer".
- **Constructors**:
  - `UserResponseDto(String)`: Constructor to initialize the token and set the token type to "Bearer".
  - `UserResponseDto()`: Default no-argument constructor.

### PseudoCode
```java
import com.path.to.user.Role;
import java.time.Instant;
import java.util.List;

@Data
public class UserResponseDto {
    private Long id;
    private String username;
    private String email;
    private List<Role> role;
    private Instant createdAt;
    private String token;
    private String tokenType;

    public UserResponseDto(String token) {
        this.token = token;
        this.tokenType = "Bearer";
    }

    public UserResponseDto() {
    }
}
```
[Back to Top](#top)


---

### UserRegistrationResponseDto

The UserRegistrationDto is used to transfer user registration data.

### Description
- **Imports**: Required classes for validation and Lombok
- **Annotations**:
    - `@Data` from Lombok to generate getters, setters, and other utility methods.
    - `@NotBlank` to ensure the field is not null and not empty.
    - `@Email` to validate the email format.
    - `@Size` to validate the size of the string.
- **Fields**:
    - `username`: The username chosen by the user, must be between 3 and 20 characters and not blank.
    - `email`: The user's email address, must be a valid email format, not blank, and not more than 50 characters.
    - `password`: The user's password, must be between 6 and 40 characters and not blank.
    - `trainerId`: Optional identifier linking the user to a specific trainer.

### PseudoCode
```java
import jakarta.validation.constraints.*;
import lombok.*;

@Data
public class UserRegistrationDto {

    @NotBlank
    @Size
    private String username;

    @NotBlank
    @Size
    private String email;

    @NotBlank
    @Size
    private String password;

    private String trainerId;
}
```
[Back to Top](#top)


---
### UserLoginResponseDto

The UserLoginDto is used to transfer user login data.

### Description
- **Imports**: Required classes for validation and Lombok
- **Annotations**:
    - `@Data` from Lombok to generate getters, setters, and other utility methods.
    - `@NotBlank` to ensure the fields are not null and not empty.
- **Fields**:
  - `username`: The user's username, must not be blank.
  - `password`: The user's password, must not be blank.

### PseudoCode
```java
import jakarta.validation.constraints.*;
import lombok.*;

@Data
@NotBlank
public class UserLoginDto {


    private String username;

    private String password;
}
```
[Back to Top](#top)

---


### UserDeleteResponseDto

The UserDeleteResponseDto is used to transfer the response message after a user deletion operation.

### Description
- **Imports**: Required classes for Lombok
- **Annotations**:
    - `@Data` from Lombok to generate getters, setters, and other utility methods.
- **Fields**:
  - `message`: The response message indicating the result of the deletion operation.

### PseudoCode
```java
package com.evoluir.evoluirfitness.dto.user;

import lombok.Data;

@Data
public class UserDeleteResponseDto {
    private String message;

    public UserDeleteResponseDto(String message) {
        this.message = message;
    }
}
```

[Back to Top](#top)

---
### UserProfileResponseDto

The UserProfileResponseDto is used to transfer user profile data.

### Description
- **Imports**: Required classes for validation and Lombok
- **Annotations**:
    - `@Data` from Lombok to generate getters, setters, and other utility methods.
- **Fields**:
  - `id`: The ID of the user profile.
  - `firstName`: The user's first name.
  - `lastName`: The user's last name.
  - `age`: The user's age.
  - `weight`: The user's weight.
  - `height`: The user's height.
  - `goals`: The user's goals.

### PseudoCode
```java
package com.evoluir.evoluirfitness.dto.user;

import lombok.Data;

import java.util.List;

@Data
public class UserProfileResponseDto {
    private Long id;
    private String firstName;
    private String lastName;
    private int age;
    private float weight;
    private float height;
    private List<String> goals;

    public UserProfileResponseDto(Long id, String firstName, String lastName, int age, float weight, float height, List<String> goals) {
        this.id = id;
        this.firstName = firstName;
        this.lastName = lastName;
        this.age = age;
        this.weight = weight;
        this.height = height;
        this.goals = goals;
    }
}
```

[Back to Top](#top)

---

### UserLogoutDto

The `UserLogoutDto` is used to transfer the response message after a user logout operation.

### Description
- **Imports**: Required classes for Lombok
- **Annotations**:
  - `@Data` from Lombok to generate getters, setters, and other utility methods.
- **Fields**:
  - `username`: The username of the user who is logging out.
  - `message`: The response message indicating the result of the logout operation.

### PseudoCode
```java
package com.evoluir.evoluirfitness.user.dto;

import lombok.Data;

@Data
public class UserLogoutDto {
    private String username;
    private final String message;

    public UserLogoutDto(String username, String message) {
        this.username = username;
        this.message = message;
    }

    public UserLogoutDto(String message) {
        this.message = message;
    }
}
```
[Back to Top](#top)

---

### UserLogoutRequestDto

The `UserLogoutRequestDto` is used to transfer the request data for a user logout operation.

### Description
- **Imports**: Required classes for Lombok
- **Annotations**:
  - `@Data` from Lombok to generate getters, setters, and other utility methods.
- **Fields**:
  - `username`: The username of the user who is logging out.

### PseudoCode
```java
package com.evoluir.evoluirfitness.user.dto;

import lombok.Data;

@Data
public class UserLogoutRequestDto {
    private String username;
}
```

[Back to Top](#top)

---

### TokenAuthenticationResponseDto

The `TokenAuthenticationResponseDto` is used to transfer the JWT access and refresh tokens in response to an authentication request.

### Description
- **Imports**: Required classes for Lombok.
- **Annotations**:
  - `@Data` from Lombok to generate getters, setters, and other utility methods.
- **Fields**:
  - `accessToken`: The JWT access token.
  - `refreshToken`: The JWT refresh token.
- **Constructor**:
  - `TokenAuthenticationResponseDto(String accessToken, String refreshToken)`: Initializes the DTO with the provided access and refresh tokens.

### PseudoCode
```java
package com.evoluir.evoluirfitness.token.dto;

import lombok.Data;

@Data
public class TokenAuthenticationResponseDto {
    private String accessToken;
    private String refreshToken;

    public TokenAuthenticationResponseDto(String accessToken, String refreshToken) {
        this.accessToken = accessToken;
        this.refreshToken = refreshToken;
    }
}
```
[Back to Top](#top)

---

### TokenRefreshRequestDto

The `TokenRefreshRequestDto` is used to transfer the refresh token in a request to obtain a new access token.

### Description
- **Imports**: Required classes for Lombok.
- **Annotations**:
  - `@Data` from Lombok to generate getters, setters, and other utility methods.
- **Fields**:
  - `refreshToken`: The JWT refresh token.

### PseudoCode
```java
package com.evoluir.evoluirfitness.token.dto;

import lombok.Data;

@Data
public class TokenRefreshRequestDto {
    private String refreshToken;
}
```

[Back to Top](#top)

---

## Security
The security configuration setup includes the main security configuration class, a utility class for handling JWT tokens, and a filter for JWT authentication. This setup ensures that user authentication and authorization are managed securely and efficiently.

---

### Security Configuration

The `SecurityConfig` class configures the security settings for the application, including JWT authentication.

### Description
- **Imports**: The necessary Spring Security and configuration classes are imported.
- **Annotations**:
  - `@Configuration` marks the class as a source of bean definitions.
  - `@EnableWebSecurity` enables Spring Security's web security support.
- **Fields**:
  - `JwtAuthEntryPoint`: Injected to handle unauthorized access attempts.
  - `CustomUserDetailsService`: Injected to manage user details.
  - `TokenService`: Injected to handle JWT token operations.
- **Constructor**:
  - Autowired constructor to inject dependencies.
- **Security Filter Chain**: The `securityFilterChain` method configures the HTTP security settings:
  - Disables CSRF protection (for simplicity, enable and configure it for production).
  - Sets up exception handling to use `authEntryPoint`.
  - Configures session management to be stateless.
  - Specifies that requests to `/api/user/auth/**` are permitted without authentication, while all other requests require authentication.
  - Adds the `JwtAuthenticationFilter` before the `UsernamePasswordAuthenticationFilter` to handle JWT token validation.
- **Methods**:
  - `passwordEncoder()`: Returns a BCrypt password encoder.
  - `authenticationManagerBean()`: Exposes the `AuthenticationManager` bean.
  - `jwtAuthenticationFilter()`: Returns a JWT authentication filter.

### PseudoCode
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    private JwtAuthEntryPoint authEntryPoint;
    private CustomUserDetailsService userDetailsService;
    private TokenService tokenService;

    @Autowired
    public SecurityConfig(CustomUserDetailsService userDetailsService, JwtAuthEntryPoint authEntryPoint, TokenService tokenService) {
        this.userDetailsService = userDetailsService;
        this.authEntryPoint = authEntryPoint;
        this.tokenService = tokenService;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        // Implementation
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        // Implementation
    }

    @Bean
    public JwtAuthenticationFilter authenticationJwtTokenFilter() {
        // Implementation
    }

    @Bean
    public AuthenticationManager authenticationManagerBean() throws Exception {
        // Implementation
    }
}
```

[Back to Top](#top)


---

Here's the formatted document with the methods included in the pseudocode:

### Token Service

The `TokenService` class handles operations related to JSON Web Tokens (JWT), such as token creation, validation, and parsing.

### Methods:

- **generateJwtToken(Authentication)**: Generates a JWT token based on the provided authentication details. Uses `SecurityConstants.JWT_SECRET` and `SecurityConstants.JWT_EXPIRATION` for token creation.
  
- **buildToken(String, Long)**: Builds a JWT token with the given username and expiration time.
  
- **generateRefreshJwtToken(String)**: Generates a JWT refresh token based on the provided username. Uses `SecurityConstants.JWT_REFRESH_EXPIRATION` for token creation.
  
- **validateAccessToken(String)**: Validates the provided JWT token using the secret key from `SecurityConstants`.

- **validateRefreshToken(String)**: Validates the provided JWT refresh token by checking its expiry date, revocation status, and the associated username.

- **getUserNameFromJwtToken(String)**: Extracts the username from the JWT token using the secret key from `SecurityConstants`.

- **createRefreshToken(String, UserEntity)**: Creates a refresh token and associates it with the user.

- **deleteRefreshToken(String)**: Deletes the refresh token from the repository based on the token string.

- **deleteAllUserTokens(String)**: Deletes all tokens associated with a specific token string from the repository.

- **deleteByUsername(String)**: Deletes all tokens associated with the given username from the repository.

- **findUsernameByToken(String)**: Finds the username associated with the given token from the repository.

- **revokeAllUserTokens(Long)**: Revokes all valid tokens associated with the specified user ID.

- **isTokenExpired(String)**: Checks if the given token is expired.

- **extractExpiration(String)**: Extracts the expiration date from the given token.

- **extractUserIdFromToken(String)**: Extracts the user ID from the given token.

- **isTokenValid(String, CustomUserDetailsService)**: Validates the token against the provided user details service to ensure the token is not expired and the username matches.

- **extractAllClaims(String)**: Extracts all claims from the given token.

- **findTokenByToken(String)**: Finds the token object from the repository based on the token string.

- **saveUserToken(UserEntity, String)**: Saves a user token in the repository after creating a refresh token.

### PseudoCode
```java
@Component
public class TokenService {

    public String generateJwtToken(Authentication authentication) {
        
    }

    private String buildToken(String username, Long expiration) {
        
    }

    public String generateRefreshJwtToken(String username) {
        
    }

    public boolean validateAccessToken(String authToken) {
        
    }

    public boolean validateRefreshToken(String token) {
        
    }

    public String getUserNameFromJwtToken(String token) {
        
    }

    public Token createRefreshToken(String jwtRefreshToken, UserEntity user) {
        // Implementation
    }

    @Transactional
    public void deleteRefreshToken(String token) {
        // Implementation
    }

    @Transactional
    public void deleteAllUserTokens(String token) {
        // Implementation
    }

    @Transactional
    public void deleteByUsername(String username) {
        // Implementation
    }

    public String findUsernameByToken(String token) {
        // Implementation
    }

    @Transactional
    public void revokeAllUserTokens(Long userId) {
        // Implementation
    }

    private boolean isTokenExpired(String token) {
        // Implementation
    }

    private Date extractExpiration(String token) {
        // Implementation
    }

    public Long extractUserIdFromToken(String token) {
        // Implementation
    }

    private boolean isTokenValid(String token, CustomUserDetailsService customUserDetailsService) {
        // Implementation
    }

    private Claims extractAllClaims(String token) {
        // Implementation
    }

    public Token findTokenByToken(String token) {
        // Implementation
    }

    public void saveUserToken(UserEntity user, String jwtToken) {
        
    }
}
```
[Back to Top](#top)


---

### JwtAuthEntryPoint

The `JwtAuthEntryPoint` class handles unauthorized access attempts by sending an `SC_UNAUTHORIZED` response.

### Explanation
- **Imports**: The necessary Jakarta Servlet and Spring Security classes.
- **Annotations**: 
  - `@Component` to mark the class as a Spring-managed bean.
- **Methods**:
  - `commence(HttpServletRequest, HttpServletResponse, AuthenticationException)`: This method is triggered whenever an unauthorized user tries to access a protected resource. It sends an `SC_UNAUTHORIZED` (401) response with the exception message. 

### PseudoCode
```java
@Component
public class JwtAuthEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest, HttpServletResponse,
                         AuthenticationException) throws IOException, ServletException {
                            
    }
}
```
[Back to Top](#top)


---


### JwtAuthenticationFilter

The `JwtAuthenticationFilter` class is a custom filter that intercepts requests to validate the JWT token.

### Description
- **Imports**: Required classes for security and JWT handling from Jakarta and Spring frameworks.
- **Annotations**:
    - `@Component` to register the class as a Spring bean.
- **Fields**:
  - `TokenService`: Utility class for JWT operations.
  - `CustomUserDetailsService`: Service to load user details.
- **Constructor**: Initializes `TokenService` and `CustomUserDetailsService`.
    - `doFilterInternal(HttpServletRequest, HttpServletResponse, FilterChain)`: Main filtering logic. Validates the JWT token and sets the authentication context if valid.
    - `getJWTFromRequest(HttpServletRequest)`: Parses the JWT token from the request header. Checks if the token is present and starts with "Bearer ", then extracts and returns the token.

### PseudoCode
```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {


    private final TokenService tokenService;
    private final CustomUserDetailsService userDetailsService;

    @Autowired
    public JwtAuthenticationFilter(TokenService, CustomUserDetailsService) {

    }

    @Override
    protected void doFilterInternal(HttpServletRequest,
                                    HttpServletResponse,
                                    FilterChain)
            throws ServletException, IOException {

    }

    private String getJWTFromRequest(HttpServletRequest) {
      
    }
}
```

[Back to Top](#top)


---

### CustomUserDetailsService

The `CustomUserDetailsService` class implements the `UserDetailsService` interface to provide user details based on the username.

### Description
- **Imports**: Required classes for user details service and Spring security.
- **Annotations**: 
  - `@Service` to mark the class as a Spring service bean.
- **Fields**:
  - `UserRepository userRepository`: Repository to access user data from the database.
- **Constructor**: Initializes `UserRepository`.
- **Methods**:
  - `loadUserByUsername(String username)`: 
    - Fetches the user by username from the repository.
    - Throws `UsernameNotFoundException` if the user is not found.
    - Returns a `User` object containing the username, password, and authorities (roles) of the user.
  - `mapRolesToGrantedAuthorities(List<Role> userRoles)`:
    - Converts the list of user roles to a collection of `GrantedAuthority` objects.
    - Uses `SimpleGrantedAuthority` to represent each role as a granted authority.

### PseudoCode
```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Autowired
    public CustomUserDetailsService(UserRepository userRepository) {

    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

    }

    private Collection<GrantedAuthority> mapRolesToGrantedAuthorities(List<Role> userRoles) {

    }
}
```
[Back to Top](#top)

---

### SecurityConstants

The SecurityConstants class contains constants related to security configurations, such as JWT settings.

### Description
- **Fields**:
  - `JWT_SECRET`: A string constant that stores the secret key for JWT.
  - `JWT_EXPIRATION`: A long constant that stores the expiration time for JWT in milliseconds.

### PseudoCode
```java
package com.evoluir.evoluirfitness.security;

public class SecurityConstants {
    public static final String JWT_SECRET = "your_jwt_secret_key";
    public static final long JWT_EXPIRATION = 86400000; // 1 day in milliseconds
}
```

[Back to Top](#top)

---

<a name="top"></a>