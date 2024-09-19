<a name="top"></a>
# Authentication and Authorization

## Table of Contents

- [Roles](#role-based-access-control-rbac)
- [Endpoints](#endpoints)
  - [User Registration](#user-registration)
  - [User Login](#user-login)
  - [User Update](#user-update)
  - [User Delete](#user-delete)
- [Implementation](#implementation)




## Role-Based Access Control (RBAC)
There are two roles in the application: a User role and a Trainer role. In this section, we will explore how the implementation of both these roles is carried out.

- **User Role**: Can book sessions, access their workout and nutrition plans, participate in virtual training, track progress, and communicate with their trainer.
- **Trainer Role**: Can manage clients, create and update workout and nutrition plans, conduct virtual training sessions, and track client progress.

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


### Implementation
For detailed overview of implementation, please refer to the documentation  [here](user_implementation.md)

[Back to Top](#top)