<a name="top"></a>
# Authentication and Authorization Implementation

## Table of Contents
- [Overview](#overview)
- [User](#user)
    - [User Entity](#user-entity)
    - [User Repository](#user-repository)
    - [User Service](#user-service)
    - [Admin Service](#adminservice)
    - [Trainer Service](#trainerservice)
    - [User Controller](#usercontroller)
    - [Admin Controller](#admincontroller)
    - [Trainer Controller](#trainercontroller)
    - [User Profile](#user-profile)
    - [Role](#role)

- [DTOs (Data Transfer Object)](#dtos-data-transfer-object)
  - [Admin](#admin)
  - [Trainer](#trainer)
  - [User](#user-1)

- [Security](#security)
  - [AuthenticationConfig](#authenticationconfig)
  - [SecurityConfig](#securityconfig)
  - [JwtAuthenticationFilter](#jwtauthenticationfilter)
  - [AuthenticateService](#authenticateservice)
  - [JwtTokenService](#jwttokenservice)
  - [LogoutService](#logoutservice)
  - [Token](#token)
  - [TokenType](#tokentype)

## Overview
This document provides a detailed high-level design for the implementation of user authentication and authorization in the fitness app. The app supports two roles: User and Trainer. Users can sign up with or without a Trainer ID. The authentication process includes user registration, login, and JWT token management using Spring Boot and AWS.

---

## User

---

### User Entity

The `UserEntity` represents the user in the database.

### Description
- **Imports**: Required classes for JPA, persistence, and Spring Security.
- **Annotations**:
  - `@Entity` to mark the class as a JPA entity.
  - `@Table` to specify the table name in the database.
  - `@Id` and `@GeneratedValue` for primary key generation.
  - `@Column` to define column properties such as nullability and uniqueness.
  - `@OneToMany` and `@OneToOne` for relationships with `Token` and `WgerToken` entities.
  - `@Enumerated` for the `role` field to store enums in the database.
  - `@Data`, `@Builder`, `@NoArgsConstructor`, and `@AllArgsConstructor` from Lombok for auto-generating boilerplate code.
- **Fields**:
  - `id`: Unique identifier for the user.
  - `username`: The username chosen by the user.
  - `firstName`: The user's first name.
  - `lastName`: The user's last name.
  - `email`: The user's email address, used as the username for authentication.
  - `password`: The user's hashed password.
  - `tokens`: A list of tokens associated with the user, mapped through a one-to-many relationship.
  - `wgerToken`: The user's token for accessing Wger API, mapped through a one-to-one relationship.
  - `role`: The user's role, enumerated as a type `Role`.
  - `trainerId`: Optional identifier linking the user to a specific trainer.
  - `accountNonExpired`, `accountNonLocked`, `credentialsNonExpired`, `enabled`: Boolean fields indicating the user's account status.
- **Methods**:
  - `getAuthorities()`: Returns the authorities granted to the user based on their role.
  - `getUsername()`: Returns the user's email as the username.
  - `getPassword()`: Returns the user's password.
  - `isAccountNonExpired()`, `isAccountNonLocked()`, `isCredentialsNonExpired()`, `isEnabled()`: Override methods to indicate the account's status.


[Back to Top](#top)

---

### User Repository

The `UserRepository` interface handles database operations for the `UserEntity`.

### Description
- **Imports**: Required classes for Spring Data JPA and `UserEntity`.
- **Annotations**: `@Repository` to mark the interface as a Spring Data repository.
- **Methods**:
  - `findByEmail()`: Custom query to find a user by their email.
  - `existsByEmail()`: Checks if a user with the given email exists.
  - `existsByUsername()`: Checks if a user with the given username exists.
  - `findByUsername()`: Custom query to find a user by their username.
  - `findById()`: Finds a user by their unique identifier.
  - `findByRoleAndTrainerId()`: Custom query to find users by their role and associated trainer ID.


[Back to Top](#top)

---

### User Service

The `UserService` handles the business logic for user registration, authentication, and management.

### Description
- **Imports**: Required classes for DTOs, models, repositories, security, logging, and Spring framework.
- **Annotations**:
  - `@Slf4j` to enable logging.
  - `@Service` to mark the class as a service.
  - `@RequiredArgsConstructor` from Lombok for automatic constructor generation with final fields.
- **Fields**:
  - `UserRepository`: Injected to manage user persistence.
  - `JwtTokenService`: Injected to handle JWT token operations.
  - `TokenRepository`: Injected to manage token persistence.
- **Methods**:
  - **`updateUser(String token, UserUpdateRequestDto request)`**: Updates an existing user's details based on the provided JWT token and request data. Validates the token, extracts the user's email, retrieves the user, updates their details, and saves the changes. Returns a `UserUpdateResponseDto` containing the updated user information.
  - **`deleteUser(String token)`**: Soft deletes a user by setting account status fields (`accountNonExpired`, `accountNonLocked`, `credentialsNonExpired`, `enabled`) to `false`. Validates the token, extracts the user's email, retrieves the user, performs the soft delete, and saves the changes. Also deletes the user from the repository.
  - **`getUserDetails(String token)`**: Retrieves the full details of a user based on the provided JWT token. Validates the token, extracts the user's email, retrieves the user, and returns a `UserDetailsResponseDto` containing the user's information.

[Back to Top](#top)

---
### AdminService

The `AdminService` handles administrative operations for managing users within the system.

### Description
- **Imports**: Required classes for repositories, services, DTOs, models, and logging.
- **Fields**:
  - `UserRepository`: Injected to handle database operations related to users.
  - `AuthenticateService`: Injected to handle user registration and authentication.
- **Methods**:
  - **`getAllUsers()`**: Retrieves a list of all users, excluding those with the `ADMIN` role. Converts each user entity to a `UserDetailsResponseDto`.
  - **`getUserByEmailAddress(UserDetailsRequestDto request)`**: Retrieves a user's details by their email address. Converts the user entity to a DTO format. Throws a `UserNotFoundException` if the user does not exist.
  - **`getUserByUserId(Long userId)`**: Retrieves a user's details by their user ID. Converts the user entity to a DTO format. Throws a `UserNotFoundException` if the user does not exist.
  - **`deleteUserById(Long userId)`**: Deletes a user by their user ID. Throws a `UserNotFoundException` if the user does not exist.
  - **`deleteUserByEmail(UserDetailsRequestDto request)`**: Deletes a user by their email address. Throws a `UserNotFoundException` if the user does not exist.
  - **`createUser(UserSignUpRequestDto user)`**: Creates a new user by reusing the `registerUser` method from `AuthenticateService`. Throws an `IllegalArgumentException` if the user already exists or if an internal error occurs.
  - **`updateUser(Long userId, UserUpdateDetailsRequestDto request)`**: Updates a user's details based on their user ID. Throws a `UserNotFoundException` if the user does not exist. Converts the updated user entity to a `UserDetailsResponseDto`.
- **Helper Methods**:
  - **`convertToDto(UserEntity user)`**: Converts a `UserEntity` to a `UserDetailsResponseDto`.

[Back to Top](#top)

---
### TrainerService

The `TrainerService` handles the business logic for managing clients assigned to trainers within the system.

### Description
- **Imports**: Required classes for repositories, services, DTOs, models, exceptions, security, and logging.
- **Fields**:
  - `UserRepository`: Injected to handle database operations related to users.
  - `AuthenticateService`: Injected to handle user registration and authentication.
- **Methods**:
  - **`getAllClients()`**: Retrieves all clients assigned to the currently authenticated trainer. Extracts the trainer's email from the security context to identify the trainer's ID and fetches all clients assigned to them. Throws a `UserNotFoundException` if the trainer is not found.
  - **`getClientDetails(Long clientId)`**: Retrieves the details of a client by their ID, verifying that the client is assigned to the currently authenticated trainer. Throws a `UserNotFoundException` if the client does not exist or is not assigned to the trainer.
  - **`deleteClient(Long clientId)`**: Deletes a client from the trainer's list by setting the client's `trainerId` to null. Verifies that the client is assigned to the authenticated trainer. Throws a `UserNotFoundException` if the client does not exist or is not assigned to the trainer.
  - **`createClient(UserSignUpRequestDto user)`**: Creates a new client, setting the `trainerId` to the ID of the currently authenticated trainer. Reuses the `registerUser` method from `AuthenticateService` for client creation. Throws an `IllegalArgumentException` if the client already exists.
  - **`addExistingClient(TrainerCreateClientRequestDto request)`**: Adds an existing client to the trainer by setting the client's `trainerId`, provided the client does not already have a trainer assigned. Throws an `IllegalArgumentException` if the client is already assigned to a trainer. Throws a `UserNotFoundException` if the client does not exist.
- **Helper Methods**:
  - **`convertToDto(UserEntity user)`**: Converts a `UserEntity` to a `UserDetailsResponseDto`.

[Back to Top](#top)

---

### UserController

The `UserController` exposes endpoints for user registration, authentication, and user management.

### Description
- **Imports**: Required classes for DTOs, models, services, Spring MVC, and logging.
- **Annotations**:
  - `@Slf4j` to enable logging.
  - `@RestController` to mark the class as a REST controller.
  - `@RequestMapping` to define the base URL for the endpoints.
  - `@RequestBody` to bind the incoming HTTP request body to the method parameter.
  - `@PostMapping`, `@PutMapping`, `@DeleteMapping`, and `@GetMapping` to map HTTP requests to handler methods.
- **Fields**:
  - `UserService`: Injected to handle user operations.
- **Methods**:
  - **`updateUser(@RequestHeader("Authorization") String token, @RequestBody UserUpdateRequestDto request)`**: Handles HTTP PUT requests to update user information. Validates the JWT token and calls the `userService.updateUser` method to perform the update. Returns a `ResponseEntity` containing the updated user information or an error message.
  - **`deleteUser(@RequestHeader("Authorization") String token)`**: Handles HTTP DELETE requests to delete a user. Validates the JWT token and calls the `userService.deleteUser` method to perform a soft delete. Returns a `ResponseEntity` containing a success message or an error message.
  - **`getUserDetails(@RequestHeader("Authorization") String token)`**: Handles HTTP GET requests to retrieve user details. Validates the JWT token and calls the `userService.getUserDetails` method. Returns a `ResponseEntity` containing the user's details or an error message.

[Back to Top](#top)

---

### AdminController

The `AdminController` exposes endpoints for admin operations, including managing users within the system.

### Description
- **Imports**: Required classes for DTOs, models, services, Spring MVC, security, and logging.
- **Annotations**:
  - `@Slf4j` to enable logging.
  - `@RestController` to mark the class as a REST controller.
  - `@RequestMapping` to define the base URL for the endpoints.
  - `@PreAuthorize` to enforce security based on roles and permissions.
  - `@GetMapping`, `@PostMapping`, `@PutMapping`, and `@DeleteMapping` to map HTTP requests to handler methods.
- **Fields**:
  - `AdminService`: Injected to handle admin-specific operations.
- **Methods**:
  - **`getAllUsers()`**: Retrieves a list of all users. Requires the `admin:read` permission.
  - **`getUserByEmailAddress(UserDetailsRequestDto request)`**: Retrieves user details by email address. Returns a not found response if the user does not exist. Requires the `admin:read` permission.
  - **`getUserByUserId(Long userId)`**: Retrieves user details by user ID. Returns a not found response if the user does not exist. Requires the `admin:read` permission.
  - **`deleteUser(Long userId)`**: Deletes a user by their ID. Returns a not found response if the user does not exist. Requires the `admin:delete` permission.
  - **`deleteUser(UserDetailsRequestDto request)`**: Deletes a user by their email address. Returns a not found response if the user does not exist. Requires the `admin:delete` permission.
  - **`createUser(UserSignUpRequestDto user)`**: Creates a new user. Returns a conflict response if the user already exists. Requires the `admin:create` permission.
  - **`updateUser(Long userId, UserUpdateDetailsRequestDto request)`**: Updates a user's details by their ID. Returns a not found response if the user does not exist. Requires the `admin:update` permission.

[Back to Top](#top)

---

### TrainerController

The `TrainerController` exposes endpoints for trainer operations, allowing trainers to manage their clients within the system.

### Description
- **Imports**: Required classes for DTOs, models, services, Spring MVC, security, and logging.
- **Annotations**:
  - `@Slf4j` to enable logging.
  - `@RestController` to mark the class as a REST controller.
  - `@RequestMapping` to define the base URL for the endpoints.
  - `@PreAuthorize` to enforce security based on roles and permissions.
  - `@GetMapping`, `@PostMapping`, and `@DeleteMapping` to map HTTP requests to handler methods.
- **Fields**:
  - `TrainerService`: Injected to handle trainer-specific operations.
- **Methods**:
  - **`getAllClients()`**: Retrieves all clients assigned to the currently authenticated trainer. Requires the `trainer:read` permission.
  - **`getClientDetails(Long clientId)`**: Retrieves the details of a client assigned to the currently authenticated trainer by their client ID. Requires the `trainer:read` permission.
  - **`deleteClient(Long clientId)`**: Deletes a client from the trainer's list of clients by the client ID. Requires the `trainer:delete` permission.
  - **`createClient(UserSignUpRequestDto user)`**: Creates a new client and assigns the currently authenticated trainer as the trainer. Requires the `trainer:create` permission.
  - **`addExistingClient(TrainerCreateClientRequestDto email)`**: Adds an existing client to the trainer's list by setting the client's `trainerId`, provided the client is not already assigned to another trainer. Requires the `trainer:create` permission.

[Back to Top](#top)

---

### UserProfile

The `UserProfile` entity represents the profile information of a user in the database.


[Back to Top](#top)

---

### Role

The `Role` enum defines different roles a user can have within the system and their associated permissions.

### Description
- **Enum Constants**:
  - `CLIENT`: Represents a regular client with no specific permissions.
  - `TRAINER`: Represents a trainer with permissions to read, create, delete, and update resources.
  - `ADMIN`: Represents an administrator with all permissions, including those of a trainer.
- **Fields**:
  - `permissions`: A set of permissions associated with the role.
- **Methods**:
  - **`getAuthorities()`**: Returns a list of `SimpleGrantedAuthority` objects representing the permissions for the role, including the role itself.

[Back to Top](#top)

---

## DTOs (Data Transfer Object)
DTOs are used to encapsulate data and transfer it from one part of an application to another.

---

### Admin

- **`UserDetailsRequestDto`**: Used to encapsulate data for user detail requests by email.
  - **Fields**:
    - `email`: The email address of the user.
  - **Methods**:
    - `toString()`: Returns a string representation of the `UserDetailsRequestDto`.

- **`UserUpdateDetailsRequestDto`**: Used to encapsulate data for updating user details.
  - **Fields**:
    - `username`: The username of the user.
    - `firstName`: The first name of the user.
    - `lastName`: The last name of the user.
    - `email`: The email address of the user.
    - `role`: The role assigned to the user.
    - `trainerId`: The ID of the trainer associated with the user.
  - **Annotations**:
    - `@Data`: Generates getters and setters for all fields.
    - `@Builder`: Provides a builder pattern for creating instances.
    - `@AllArgsConstructor`: Generates a constructor with all fields.
    - `@NoArgsConstructor`: Generates a no-argument constructor.


[Back to Top](#top)

---

### Trainer

- **`TrainerCreateClientRequestDto`**: Used to encapsulate data for creating or adding an existing client to a trainer.
  - **Fields**:
    - `email`: The email address of the client to be created or added.
  - **Annotations**:
    - `@Data`: Generates getters and setters for all fields.
    - `@Builder`: Provides a builder pattern for creating instances.
    - `@AllArgsConstructor`: Generates a constructor with all fields.
    - `@NoArgsConstructor`: Generates a no-argument constructor.

[Back to Top](#top)

---

### User

- **`UserDetailsResponseDto`**: Used to encapsulate data for user details responses.
  - **Fields**:
    - `username`: The username of the user.
    - `firstName`: The first name of the user.
    - `lastName`: The last name of the user.
    - `email`: The email address of the user.
    - `trainerId`: The ID of the trainer associated with the user.
    - `role`: The role assigned to the user.
  - **Annotations**:
    - `@Data`: Generates getters and setters for all fields.
    - `@Builder`: Provides a builder pattern for creating instances.
    - `@AllArgsConstructor`: Generates a constructor with all fields.


- **`UserLoginRequestDto`**: Used to encapsulate data for user login requests.
  - **Fields**:
    - `email`: The email address of the user.
    - `password`: The password of the user.
  - **Annotations**:
    - `@Data`: Generates getters and setters for all fields.
    - `@Builder`: Provides a builder pattern for creating instances.
    - `@AllArgsConstructor`: Generates a constructor with all fields.
    - `@NoArgsConstructor`: Generates a no-argument constructor.


- **`UserLoginResponseDto`**: Used to encapsulate data for user login responses.
  - **Fields**:
    - `email`: The email address of the user.
    - `accessToken`: The access token for the user, mapped to `access_token` in JSON.
    - `refreshToken`: The refresh token for the user, mapped to `refresh_token` in JSON.
  - **Annotations**:
    - `@Data`: Generates getters and setters for all fields.
    - `@Builder`: Provides a builder pattern for creating instances.
    - `@AllArgsConstructor`: Generates a constructor with all fields.
    - `@NoArgsConstructor`: Generates a no-argument constructor.
    

- **`UserLogoutResponseDto`**: Used to encapsulate data for user logout responses.
  - **Fields**:
    - `message`: The message indicating the logout status.
  - **Annotations**:
    - `@Data`: Generates getters and setters for all fields.
    - `@Builder`: Provides a builder pattern for creating instances.
    - `@AllArgsConstructor`: Generates a constructor with all fields.
    - `@NoArgsConstructor`: Generates a no-argument constructor.


- **`UserSignUpRequestDto`**: Used to encapsulate data for user sign-up requests.
  - **Fields**:
    - `username`: The username of the new user.
    - `firstName`: The first name of the new user.
    - `lastName`: The last name of the new user.
    - `email`: The email address of the new user.
    - `password`: The password of the new user.
    - `role`: The role assigned to the new user.
    - `trainerId`: The ID of the trainer associated with the new user.
  - **Annotations**:
    - `@Data`: Generates getters and setters for all fields.
    - `@Builder`: Provides a builder pattern for creating instances.
    - `@AllArgsConstructor`: Generates a constructor with all fields.
    - `@NoArgsConstructor`: Generates a no-argument constructor.


- **`UserSignUpResponseDto`**: Used to encapsulate data for user sign-up responses.
  - **Fields**:
    - `username`: The username of the new user.
    - `firstName`: The first name of the new user.
    - `lastName`: The last name of the new user.
    - `email`: The email address of the new user.
    - `role`: The role assigned to the new user.
    - `trainerId`: The ID of the trainer associated with the new user.
    - `accessToken`: The access token for the new user, mapped to `access_token` in JSON.
    - `refreshToken`: The refresh token for the new user, mapped to `refresh_token` in JSON.
  - **Annotations**:
    - `@Data`: Generates getters and setters for all fields.
    - `@Builder`: Provides a builder pattern for creating instances.
    - `@AllArgsConstructor`: Generates a constructor with all fields.
    - `@NoArgsConstructor`: Generates a no-argument constructor.


- **`UserUpdateRequestDto`**: Used to encapsulate data for updating user details.
  - **Fields**:
    - `firstName`: The first name of the user.
    - `lastName`: The last name of the user.
    - `email`: The email address of the user.
    - `trainerId`: The ID of the trainer associated with the user.
  - **Annotations**:
    - `@Data`: Generates getters and setters for all fields.


- **`UserUpdateResponseDto`**: Used to encapsulate data for user update responses.
  - **Fields**:
    - `username`: The username of the updated user.
    - `firstName`: The first name of the updated user.
    - `lastName`: The last name of the updated user.
    - `email`: The email address of the updated user.
    - `trainerId`: The ID of the trainer associated with the updated user.
  - **Annotations**:
    - `@Data`: Generates getters and setters for all fields.
    - `@Builder`: Provides a builder pattern for creating instances.

[Back to Top](#top)

---

## Security

The security module contains the essential components responsible for managing authentication, authorization, and access control within the application. This setup ensures that only authenticated users with the correct roles and permissions can access protected resources.

---

### AuthenticationConfig

- **`AuthenticationConfig`**: Configures the application's authentication mechanisms.
  - **Components**:
    - **`UserDetailsService`**: Retrieves user details from the `UserRepository` to validate user credentials.
    - **`AuthenticationProvider`**: Sets up a `DaoAuthenticationProvider` to handle user authentication, using the custom `UserDetailsService` and a password encoder.
    - **`AuthenticationManager`**: Configures the `AuthenticationManager` to handle authentication requests.
    - **`PasswordEncoder`**: Uses `BCryptPasswordEncoder` to securely encode passwords.
  - **Annotations**:
    - `@Configuration`: Marks this class as a configuration class for Spring.
    - `@RequiredArgsConstructor`: Generates a constructor with required fields for dependency injection.

[Back to Top](#top)

---

### SecurityConfig

- **`SecurityConfig`**: Defines the core security settings for the application.
  - **Components**:
    - **`securityFilterChain`**: Configures the HTTP security, including:
      - Permitting requests to specific endpoints and requiring authentication for all other requests.
      - Managing sessions as stateless using `SessionCreationPolicy.STATELESS`.
      - Adding a custom JWT filter (`jwtAuthFilter`) to the security filter chain.
      - Specifying a custom `AccessDeniedHandler` to handle access-denied situations.
      - Configuring logout behavior and a logout handler.
    - **`AccessDeniedHandler`**: Custom handler that returns a JSON response with a 403 status and a message when a user is denied access to a resource.
  - **Annotations**:
    - `@Configuration`: Indicates that this class provides Spring Security configuration.
    - `@EnableWebSecurity`: Enables web security in the application.
    - `@EnableMethodSecurity`: Allows method-level security, such as securing specific methods using annotations.
    - `@RequiredArgsConstructor`: Generates a constructor for dependency injection with required fields.

[Back to Top](#top)

---
### JwtAuthenticationFilter

- **`JwtAuthenticationFilter`**: A Spring Security filter that intercepts each HTTP request to validate the JWT token provided in the Authorization header.
  - **Components**:
    - **`JwtTokenService`**: Service to handle JWT token creation, validation, and extraction of user information.
    - **`UserDetailsService`**: Used to load user details from the database using the extracted user email from the JWT.
    - **`TokenRepository`**: Interacts with the database to validate and manage refresh tokens.
    - **`SecurityTokenProperties`**: Contains properties for handling security configurations, such as identifying routes that require a refresh token.
  - **Key Operations**:
    - Extracts the JWT token from the `Authorization` header.
    - Checks if the user email can be extracted from the token.
    - Verifies if the token is valid and not expired or revoked.
    - If valid, creates an authentication token and sets it in the `SecurityContext` for the request to proceed.
    - Handles different types of tokens (access and refresh) based on the current request's URI.
    - Clears the security context and sends an appropriate error response if the token is invalid or an error occurs during processing.
  - **Helper Methods**:
    - **`isRefreshTokenRequired(String requestURI)`**: Determines if the current request requires a refresh token based on its URI.
    - **`handleAuthError(HttpServletResponse response, int statusCode, String errorMessage)`**: Sends a JSON response with the appropriate status code and error message in case of authentication errors.
  - **Annotations**:
    - `@Component`: Marks this class as a Spring component, allowing it to be automatically detected and managed by Spring.
    - `@Slf4j`: Enables logging.
    - `@RequiredArgsConstructor`: Generates a constructor for dependency injection with required fields.

[Back to Top](#top)

---

### AuthenticateService

- **`AuthenticateService`**: Handles the core authentication and user management operations within the application, such as registration, login, logout, and token handling.

  - **Components**:
    - **`UserRepository`**: Interacts with the database to manage user information.
    - **`PasswordEncoder`**: Encodes passwords securely.
    - **`JwtTokenService`**: Manages JWT creation, validation, and extraction of user information.
    - **`AuthenticationManager`**: Authenticates user login attempts.
    - **`TokenRepository`**: Stores and manages user tokens (both access and refresh).
    - **`WgerAuthService`**: Manages interactions with an external Wger service for user registration and login.

  - **Key Methods**:
    - **`registerUser(UserSignUpRequestDto request)`**: Registers a new user by validating the input data, checking for existing users, creating a user entity, saving the user to the database, generating JWT and refresh tokens, saving the tokens, and optionally registering the user with the Wger service. Throws `UserAlreadyExistsException` if the user already exists.
    - **`authenticateUser(UserLoginRequestDto request)`**: Authenticates a user using the provided email and password, generates JWT and refresh tokens upon successful authentication, and saves the tokens to the database.
    - **`logoutAllUserSessions(String token)`**: Logs out the user from all active sessions by invalidating all tokens associated with the user. Throws `UserNotFoundException` if the user is not found.
    - **`refreshToken(HttpServletRequest request, HttpServletResponse response)`**: Refreshes the JWT token if the provided refresh token is valid and not expired. Returns a new access token in the response.
    - **`handleAuthError(HttpServletResponse response, int statusCode, String errorMessage)`**: Sends an error response with the specified status code and message in case of authentication errors.

  - **Helper Methods**:
    - **`validateRequest(UserSignUpRequestDto request)`**: Checks if all required fields are present in the registration request and validates the email format and password strength.
    - **`isNullOrEmpty(String str)`**: Checks if a string is null or empty.
    - **`isStrongPassword(String password)`**: Validates if the provided password meets the required strength criteria.
    - **`isValidEmailFormat(String email)`**: Validates the format of an email address.
    - **`checkIfUserExists(String email, String username)`**: Checks if a user with the provided email or username already exists.
    - **`determineUserRole(String role)`**: Determines the user's role, defaulting to `CLIENT` if no role is provided.
    - **`createUserEntity(UserSignUpRequestDto request, Role role)`**: Creates a new `UserEntity` with the provided details and the determined role.
    - **`saveUserToken(UserEntity savedUser, String jwtToken)`**: Saves the user's token in the database.
    - **`invalidateAllUserTokens(UserEntity user)`**: Invalidates all valid tokens associated with the user by setting them as expired and revoked.

  - **Annotations**:
    - `@Service`: Marks this class as a service in the Spring framework.
    - `@RequiredArgsConstructor`: Generates a constructor with required fields for dependency injection.
    - `@Slf4j`: Enables logging.

[Back to Top](#top)

---
### JwtTokenService

- **`JwtTokenService`**: Provides utility methods for handling JWT token creation, validation, extraction, and expiration.
  - **Components**:
    - **`TokenRepository`**: Interacts with the database to manage tokens.
    - **`privateKey` & `publicKey`**: Injected from application properties to sign and validate JWT tokens using RSA encryption.
    - **`jwtTokenExpiration` & `refreshTokenExpiration`**: Configurable properties that define the lifespan of access and refresh tokens.
  - **Key Methods**:
    - **`extractUsername(String jwtToken)`**: Extracts the username (email) from a JWT token.
    - **`generateToken(UserDetails userDetails)`**: Generates a JWT access token for the provided user.
    - **`generateAdminToken(UserDetails userDetails)`**: Generates a JWT token specifically for users with an admin role.
    - **`generateRefreshToken(UserDetails userDetails)`**: Generates a refresh token for the provided user.
    - **`isTokenValid(String jwtToken, UserDetails userDetails)`**: Checks if a token is valid by comparing the extracted username with the user's username and ensuring the token is not expired.
    - **`getPrivateKey()` & `getPublicKey()`**: Decodes the RSA private and public keys for signing and verifying JWT tokens.
  - **Helper Methods**:
    - **`extractClaim(String jwtToken, Function<Claims, T> claimsResolver)`**: Extracts a specific claim from the JWT token.
    - **`buildToken(Map<String, Object> extraClaims, UserDetails userDetails, long expiration)`**: Builds a new JWT token with the specified claims, user details, and expiration time.
    - **`isTokenExpired(String jwtToken)`**: Checks if a token has expired by extracting its expiration date.
    - **`extractAllClaims(String jwtToken)`**: Extracts all claims from a JWT token using the public key.
  - **Annotations**:
    - `@Service`: Marks this class as a service in the Spring framework.
    - `@RequiredArgsConstructor`: Generates a constructor with required fields for dependency injection.

[Back to Top](#top)

---

### LogoutService

- **`LogoutService`**: Handles the logout process for users, either invalidating a single token or all tokens, depending on the request URI.
  - **Components**:
    - **`TokenRepository`**: Interacts with the database to manage tokens.
    - **`UserRepository`**: Provides access to user data for validating the existence of users.
    - **`JwtTokenService`**: Provides utility methods for token validation and extraction of user details.
  - **Key Methods**:
    - **`logout(HttpServletRequest request, HttpServletResponse response, Authentication authentication)`**: Handles the logout request by invalidating the token present in the request header. Extracts the token, verifies its presence, invalidates it if valid, and builds an appropriate response.
    - **`handleTokenInvalidation(HttpServletResponse response, String jwtToken)`**: Validates and invalidates a token, logging out the user and building a logout response.
    - **`buildLogoutResponse(HttpServletResponse response, HttpStatus status, String message)`**: Constructs and writes a response to indicate the logout status and any error messages.
  - **Helper Methods**:
    - **`extractTokenFromRequest(HttpServletRequest request)`**: Extracts the JWT token from the `Authorization` header in the request.
    - **`validateAndInvalidateToken(String tokenValue)`**: Checks if the token is present, not expired, and not revoked. If valid, it marks the token as expired and revoked.
    - **`doesUserExists(String email)`**: Checks if a user with the given email exists in the database.
    - **`isTokenPresent(String token)`**: Verifies if the token is present in the database.
  - **Annotations**:
    - `@RequiredArgsConstructor`: Generates a constructor with required fields for dependency injection.

[Back to Top](#top)

---

### Token

- **`Token`**: Represents a JWT token entity in the database, used for managing access and refresh tokens associated with users.
  - **Fields**:
    - `id`: The unique identifier for the token.
    - `token`: The JWT token value, stored as a unique string with a length of up to 2000 characters.
    - `tokenType`: Specifies the type of the token (either `BEARER` or `REFRESH`).
    - `expired`: A boolean flag indicating if the token has expired.
    - `revoked`: A boolean flag indicating if the token has been revoked.
    - `user`: A many-to-one relationship with the `UserEntity`, linking the token to a specific user.
  - **Annotations**:
    - `@Entity`: Marks this class as a JPA entity.
    - `@Id` & `@GeneratedValue`: Specifies the primary key and its generation strategy.
    - `@Column`: Defines the token field as unique and sets its maximum length.
    - `@Enumerated`: Maps the `tokenType` field to an enumeration.
    - `@ManyToOne`: Establishes a many-to-one relationship with the `UserEntity`.
    - `@JoinColumn`: Specifies the foreign key column (`user_id`) in the token table to reference the associated user.

[Back to Top](#top)

---

### TokenType

- **`TokenType`**: An enumeration that defines the possible types of tokens used in the application.
  - **Enum Constants**:
    - `BEARER`: Represents a standard access token.
    - `REFRESH`: Represents a refresh token used to generate new access tokens.

[Back to Top](#top)

---