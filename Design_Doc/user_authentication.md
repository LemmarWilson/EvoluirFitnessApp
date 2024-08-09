# Authentication and Authorization

## Table of Contents
- [Implementation](#implementation)
- [Roles](#role-based-access-control-rbac)
    - [User](#user)
    - [Trainer](#trainer)
- [Endpoints](#endpoints)
  - [User Registration](#user-registration)
  - [User Login](#user-login)
  - [User Update](#user-update)
  - [User Delete](#user-delete)
  - [Get All Users](#get-all-users)
  - [Get User by ID](#get-user-by-id)
- [Token Management](#token-management)
- [Security](#security)
- [API Endpoints](#api-endpoints-summary)



## Role-Based Access Control (RBAC)
There are two roles in the application: a User role and a Trainer role. In this section, we will explore how the implementation of both these roles is carried out.

- **User Role**: Can book sessions, access their workout and nutrition plans, participate in virtual training, track progress, and communicate with their trainer.
- **Trainer Role**: Can manage clients, create and update workout and nutrition plans, conduct virtual training sessions, and track client progress.

## User

A user will be a client of the personal trainer. Users can book sessions, access workout and nutrition plans, participate in virtual training, track their progress, and communicate with their trainer. Users can also register without a trainer ID and just use the app as a health tracker.

[Back to Top](#top)

---
## Trainer

A trainer will have additional capabilities to manage their clients, create workout and nutrition plans, and conduct virtual training sessions.

[Back to Top](#top)

---

## Endpoints

#### User Registration
- **Endpoint**: `/api/user/auth/register`
- **Method**: `POST`
- **Details**: This endpoint is designed to handle incoming HTTP POST requests for user registration. The request body must contain the user's chosen username, their email address, a password, and optionally a Trainer ID if they are connecting to a specific trainer. This flexibility allows users to either join independently or start with a pre-defined trainer relationship.
- **Request Body**: 
  ```json
  {
    "username": "string",
    "email": "string",
    "password": "string",
    "trainerId": "optional string",
  }
  ```
- **Response**: 
  ```json
  {
    "id": "integer",
    "username": "string",
    "email": "string",
    "roles": ["USER"],
    "createdAt": "timestamp"
  }
  ```

[Back to Top](#top)

---

#### User Login
- **Endpoint**: `/api/user/auth/login`
- **Method**: `POST`
- **Details**: This endpoint is responsible for handling user login requests. Users must provide their username and password to authenticate. If the credentials are valid, the system responds with the user's details and an access token for subsequent authenticated requests.
- **Request Body**: 
  ```json
  {
    "username": "string",
    "password": "string"
  }
  ```
- **Response**: 
  ```json
  {
    "id": "integer",
    "username": "string",
    "email": "string",
    "roles": ["USER"],
    "accessToken": "string",
    "tokenType": "Bearer"
  }
  ```
[Back to Top](#top)

---

#### User Update
- **Endpoint**: `/api/user/update/{userId}`
- **Method**: `PUT`
- **Details**: This endpoint updates an existing user's details. Users can update their personal information such as username, email, and other profile details. The `userId` path variable identifies the user to be updated.
- **Request Body**: 
  ```json
  {
    "username": "optional string",
    "email": "optional string",
    "password": "optional string",
    "trainerId": "optional string",
  }
  ```
- **Response**: 
  ```json
  {
    "id": "integer",
    "username": "string",
    "email": "string",
    "roles": ["USER"],
    "updatedAt": "timestamp"
  }
  ```
[Back to Top](#top)

---

#### User Delete
- **Endpoint**: `/api/user/delete/{userId}`
- **Method**: `DELETE`
- **Details**: This endpoint handles the deletion of a user. The `userId` path variable identifies the user to be deleted.
- **Response**: 
  ```json
  {
    "message": "User successfully deleted"
  }
  ```
[Back to Top](#top)

---

#### Get All Users
- **Endpoint**: `/api/user`
- **Method**: `GET`
- **Details**: This endpoint retrieves a list of all users in the system.
- **Response**: 
  ```json
  [
    {
      "id": "integer",
      "username": "string",
      "email": "string",
      "roles": ["USER"],
      "createdAt": "timestamp"
    },
    ...
  ]
  ```
[Back to Top](#top)

---

#### Get User by ID
- **Endpoint**: `/api/user/admin/auth/{userId}`
- **Method**: `GET`
- **Details**: This endpoint retrieves a user by their ID. The `userId` path variable identifies the user to be retrieved. This view is only available for admin users.
- **Response**: 
  ```json
  {
    "id": "integer",
    "username": "string",
    "email": "string",
    "roles": ["USER"],
    "createdAt": "timestamp"
  }
  ```
[Back to Top](#top)

---

#### User Profile
- **Endpoint**: `/api/user/profile/{userId}`
- **Method**: `GET`
- **Details**: This endpoint allows authenticated users to retrieve their profile information. The request must include a valid access token in the Authorization header.
- **Request Headers**: 
  ```http
  Authorization: Bearer <accessToken>
  ```
- **Response**: 
  ```json
  {
    "user": {
      "id": "integer",
      "username": "string",
      "email": "string",
      "trainerId": "string",
      "createdAt": "timestamp" ,
      "role": ["string"],
      "profile": {
        "firstName": "string",
        "lastName": "string",
        "age": "integer",
        "gender": "string",
        "height": "float",
        "weight": "float",
        "goals": "[string]"
      }   
    }
  }
  ```
[Back to Top](#top)

---
#### Trainer Registration
- **Endpoint**: `/api/auth/register-trainer`
- **Method**: `POST`
- **Details**: This endpoint handles the registration of trainers. Trainers need to provide their username, email, password, and a list of certifications. This data will be used to create a new trainer profile in the system.
- **Request Body**: 
  ```json
  {
    "username": "string",
    "email": "string",
    "password": "string",
    "certifications": ["string"]
  }
  ```
- **Response**: 
  ```json
  {
    "id": "integer",
    "username": "string",
    "email": "string",
    "roles": ["TRAINER"],
    "certifications": ["string"],
    "createdAt": "timestamp"
  }
  ```

#### Trainer Login
- **Endpoint**: `/api/auth/login`
- **Method**: `POST`
- **Details**: This endpoint processes login requests for trainers. They must submit their username and password for authentication. A successful login returns the trainer's details and an access token for authenticated requests.
- **Request Body**: 
  ```json
  {
    "username": "string",
    "password": "string"
  }
  ```
- **Response**: 
  ```json
  {
    "id": "integer",
    "username": "string",
    "email": "string",
    "roles": ["TRAINER"],
    "accessToken": "string",
    "tokenType": "Bearer"
  }
  ```

#### Trainer Profile
- **Endpoint**: `/api/trainer/profile`
- **Method**: `GET`
- **Details**: This endpoint allows authenticated trainers to retrieve their profile information. A valid access token must be included in the Authorization header of the request.
- **Request Headers**: 
  ```http
  Authorization: Bearer <accessToken>
  ```
- **Response**: 
  ```json
  {
    "id": "integer",
    "username": "string",
    "email": "string",
    "roles": ["TRAINER"],
    "createdAt": "timestamp",
    "profile": {
      "firstName": "string",
      "lastName": "string",
      "certifications": ["string"],
      "experience": "integer"
    }
  }
  ```

### Token Management
- **Access Tokens**: Issued upon successful login and used for authenticating API requests.
- **Refresh Tokens**: Used to issue new access tokens without requiring the user to log in again.


### Security
- **Password Hashing**: All user passwords are hashed using a secure hashing algorithm (e.g., BCrypt) before storing in the database.
- **JWT Authentication**: Uses JSON Web Tokens (JWT) for secure user authentication and authorization.
- **HTTPS**: All API endpoints are secured using HTTPS to ensure encrypted communication.

### API Endpoints Summary
- **User Endpoints**: 
  - `/api/user/auth/register`
  - `/api/user/auth/login`
  - `/api/user/profile`
- **Trainer Endpoints**:
  - `/api/auth/register-trainer`
  - `/api/auth/login`
  - `/api/trainer/profile`

### Implementation
For detailed overview of implementation, please refer to the documentation  [here](user_implementation.md)

### High Level Design
The detailed HLD for User Authentication and Authorization can be found [here](./user_authentication_hld.md).

[Back to Top](#top)
<a name="top"></a>