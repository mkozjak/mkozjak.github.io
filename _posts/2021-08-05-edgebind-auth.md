---
layout: post
title: "User Auth pt.1: AWS authentication"
comments: false
keywords: "aws cognito terraform authentication"
---

As an introduction to User Auth series we will be looking into authenticating user profiles using JWT tokens for regular login and custom tokens generated using the HMAC algorithm leveraging technologies such as AWS Cognito, Lambda, JavaScript, Go and Terraform.

![picture](/assets/images/blog-auth-edgebind-1a.png)*A live authentication and authorization situation.*

Our goal for today would be to:
 * Register a new user
 * Login user using a Web browser
 * Login user using an API key
 * Write Terraform configuration for all the required AWS infrastructure resources
 
**Registering a new user**

To register a brand new user we need to build a new web page that'll require the use of a JavaScript library on frontend, called <a href="https://github.com/aws-amplify/amplify-js" target="_blank">amplify-js</a>. For that task I decided to go with <a href="https://stimulus.hotwired.dev" target="_blank">Stimulus</a> and Webpack. To bootstrap a fresh new project, I used stimulus-starter:
```sh
$ git clone https://github.com/stimulusjs/stimulus-starter.git
$ cd stimulus-starter
$ yarn install
$ yarn start
```

Stimulus is a minimalist JavaScript library loaded into regular HTML5 code enabling you to use its controllers based on the action needed.

Let's start by implementing our Sign up front page to prompt the user for their email and a fresh new, secure password:
```html
<h1>Sign in to Edgebind</h1>
<div class="auth-input-container">
  <input class="login-textbox" placeholder="Username" data-action="input->auth#validateInput keyup->auth#enterlogin" data-auth-target="username" type="text">
</div>
<div class="auth-input-container">
  <input class="login-textbox" placeholder="Password" data-action="input->auth#validateInput keyup->auth#enterlogin" data-auth-target="password" type="password"><br>
  <a class="forgot-password-button" href="/auth/password_reset.html">Forgot password?</a>
</div>
<div>
  <button id ="auth-login-button" class="auth-login-button auth-button-disabled" data-action="auth#login">Sign In</button>
  <a class="create-account-button" href="/auth/register">Create a New Account</a>
</div>
<div id="auth-input-incorrect" class="auth-input-incorrect">
  <p>&#9888; Wrong email or password!</p>
</div>
```

With some CSS magic applied, this code will render our sign up front page:
![picture](/assets/images/blog-auth-edgebind-1b.png)

In order to fetch some more details from our user, we need to write some more code. For example, let's look at the following code including a contact form with a button that'll run our registering Stimulus controller, `auth#register`:
```html
<h1>Contact information</h1>
<div class="auth-input-container">
  <h2>Full name</h2>
  <input type="text" class="register-textbox" data-action="input->auth#validateRegInfo" data-target="auth.fullname">
</div>
<div class="auth-input-container">
  <h2>Company name</h2>
  <input type="text" class="register-textbox" data-action="input->auth#validateRegInfo" data-target="auth.company">
</div>
<div class="auth-input-container">
  <h2>Address</h2>
  <input type="text" class="register-textbox" data-action="input->auth#validateRegInfo" placeholder="Street, Company Name, etc." data-target="auth.addr1">
  <br>
  <input type="text" class="register-textbox" placeholder="Suite, unit, building, floor, etc." data-target="auth.addr2">
</div>
<div class="auth-input-container">
  <h2>Country/Region</h2>
  <select class="tab-dropdown-sub" style="width:90%;" data-target="auth.country">
    <option value="germany">Germany</option>
    <option value="italy">Italy</option>
  </select>
</div>
<div class="auth-input-container">
  <h2>City</h2>
  <input type="text" class="register-textbox" data-action="input->auth#validateRegInfo" data-target="auth.city">
</div>
<div class="auth-input-container">
  <h2>State / Province or region</h2>
  <input type="text" class="register-textbox" data-target="auth.state">
</div>
<div class="auth-input-container">
  <h2>Postal code</h2>
  <input type="text" class="register-textbox" data-target="auth.postcode">
</div>
<button id="register-create-button" class="auth-login-button auth-button-disabled" data-action="auth#register">Create Account</button>
```

Now we're set to get our sign up details page rendered:
![picture](/assets/images/blog-auth-edgebind-1c.png)

Once the `Create a New Account` button is clicked, a function called `register` in the `auth` controller source file is called. It uses amplify-js' <a href="https://docs.amplify.aws/lib/auth/emailpassword/q/platform/js#sign-up" target="_blank">Auth.signUp</a> method to register the user to AWS Cognito User Pool:
```js
async register(event) {
    try {
        await Auth.signUp({
            username: this.username,
            password: this.password,
            attributes: {
                email: this.username,
                name: this.fullnameTarget.value.split(" ")[0],
                family_name: this.fullnameTarget.value.split(" ")[1]
                address: this.addr1Target.value + " " + this.addr2Target.value,
                "custom:city": this.cityTarget.value,
                "custom:country": this.countryTarget.value,
                "custom:company": this.companyTarget.value,
                "custom:zip_code": this.postcodeTarget.value
            }
        })
    } catch (error) {
        notify("Error signing up: " + error.message)
    }

    window.location.href = "/auth/thankyou"
}
```

If we'd want to run this implementation against a live system, we'd need to spin up cloud resources, of course. For that we will use <a href="https://www.terraform.io" target="_blank">Terraform</a> to configure and bring up all the relevant infrastructure resources:

```tf
resource "aws_cognito_user_pool" "edgebind_subscribers" {
  name                     = "edgebind-subscribers-prod"
  username_attributes      = ["email"]
  auto_verified_attributes = ["email"]
  mfa_configuration        = "OPTIONAL"

  email_configuration {
    email_sending_account = "COGNITO_DEFAULT"
  }

  software_token_mfa_configuration {
    enabled = true
  }

  admin_create_user_config {
    allow_admin_create_user_only = "false"
  }

  password_policy {
    temporary_password_validity_days = 1
    minimum_length                   = 8
    require_lowercase                = true
    require_numbers                  = true
    require_symbols                  = true
    require_uppercase                = true
  }

  lambda_config {
    custom_message = aws_lambda_function.api_getsignuplink.arn
  }

  # IMPORTANT: modify schema by editing it manually through aws console
  # and then import the changes here
  # https://github.com/terraform-providers/terraform-provider-aws/issues/3891
  schema {
    developer_only_attribute = false
    attribute_data_type      = "String"
    name                     = "zip_code"
    mutable                  = true
    required                 = false

    string_attribute_constraints {
      min_length = 1
      max_length = 50
    }
  }

  schema {
    developer_only_attribute = false
    attribute_data_type      = "String"
    name                     = "email"
    mutable                  = true
    required                 = true

    string_attribute_constraints {
      min_length = 5
      max_length = 50
    }
  }

  schema {
    developer_only_attribute = false
    attribute_data_type      = "String"
    name                     = "address"
    mutable                  = true
    required                 = true

    string_attribute_constraints {
      min_length = 2
      max_length = 50
    }
  }

  schema {
    developer_only_attribute = false
    attribute_data_type      = "String"
    name                     = "name"
    mutable                  = true
    required                 = true

    string_attribute_constraints {
      min_length = 1
      max_length = 50
    }
  }

  schema {
    developer_only_attribute = false
    attribute_data_type      = "String"
    name                     = "family_name"
    mutable                  = true
    required                 = true

    string_attribute_constraints {
      min_length = 1
      max_length = 50
    }
  }

  schema {
    developer_only_attribute = false
    attribute_data_type      = "String"
    name                     = "company"
    mutable                  = false
    required                 = false

    string_attribute_constraints {
      min_length = 1
      max_length = 50
    }
  }

  schema {
    developer_only_attribute = false
    attribute_data_type      = "String"
    name                     = "city"
    mutable                  = true
    required                 = false

    string_attribute_constraints {
      min_length = 1
      max_length = 50
    }
  }

  schema {
    developer_only_attribute = false
    attribute_data_type      = "String"
    name                     = "country"
    mutable                  = true
    required                 = false

    string_attribute_constraints {
      min_length = 1
      max_length = 50
    }
  }

  schema {
    developer_only_attribute = false
    attribute_data_type      = "String"
    name                     = "subscriber_uuid"
    mutable                  = false
    required                 = false

    string_attribute_constraints {
      min_length = 0
      max_length = 36
    }
  }

  schema {
    developer_only_attribute = false
    attribute_data_type      = "String"
    name                     = "role"
    mutable                  = true
    required                 = true

    string_attribute_constraints {
      min_length = 0
      max_length = 10
    }
  }

  # https://github.com/terraform-providers/terraform-provider-aws/issues/3891
  lifecycle {
    ignore_changes = all
  }
}

resource "aws_cognito_user_pool_client" "edgebind_subscribers" {
  name            = "edgebind-clients"
  user_pool_id    = aws_cognito_user_pool.edgebind_subscribers.id
  generate_secret = false
}
```