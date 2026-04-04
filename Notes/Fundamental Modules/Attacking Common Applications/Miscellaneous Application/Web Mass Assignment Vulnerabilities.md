## Web Mass Assignment Vulnerabilities

Mass assignment is a developer convenience feature in frameworks like Ruby on Rails, Django, Laravel, and others that allows a web application to automatically bind HTTP request parameters directly to model attributes.  When developers fail to whitelist which fields are permitted for mass assignment, an attacker who understands the underlying data model can inject extra parameters into a request and modify attributes they were never supposed to control, such as privilege flags, account status fields, or billing tiers. 
***

## How It Works

A standard registration form might POST:

```
username=john&password=pass123&email=john@example.com
```

If the backend blindly maps all request parameters to the user model, an attacker can append additional fields the form never presented:

```
username=john&password=pass123&email=john@example.com&admin=true
```

In Ruby on Rails, even if the `User` model only declares `username` and `email` as accessible, if the controller does not enforce a parameter whitelist, the `admin` attribute is still assigned because it exists in the request body:

```ruby
class User < ActiveRecord::Base
  attr_accessible :username, :email
end
```

The server receives `{ "user" => { "username" => "hacker", "email" => "hacker@example.com", "admin" => true } }` and sets `admin: true` on the new record despite it not being in the declared `attr_accessible` list in older Rails versions.

***

## Hands-On: Registration Approval Bypass

The target is an Asset Manager web application with source code available. After registering a new account normally, the application displays `Account is pending approval`, blocking login until an administrator manually enables the account.

### Reading the Source Code

Reviewing `app.py` reveals the login check queries the database for three columns per user:

```python
for i, j, k in cur.execute('select * from users where username=? and password=?', (username, password)):
    if k:
        session['user'] = i
        return redirect("/home", code=302)
    else:
        return render_template('login.html', value='Account is pending for approval')
```

The variable `k` is the third column, representing the confirmed/approved status. If `k` is falsy (i.e., `False` or `0`), login is blocked.

Reviewing the registration handler reveals how that third column gets set:

```python
try:
    if request.form['confirmed']:
        cond = True
except:
    cond = False

cur.execute('insert into users values(?,?,?)', (username, password, cond))
```

The application checks whether a `confirmed` key exists in the POST form data. If the key is present with any non-empty value, `cond` is set to `True` and the account is inserted as pre-approved. The form simply never renders the `confirmed` field, so a normal user never sends it. But there is no server-side enforcement preventing it from being sent manually.

### Exploitation with Burp Suite

1. Navigate to the registration page and fill in the form normally
2. Intercept the POST request to `/register` in Burp Suite before it is sent
3. Add the `confirmed` parameter to the request body:

```
POST /register HTTP/1.1

username=new&password=test&confirmed=test
```

4. Forward the request. The response returns `Success!!`
5. Log in with `new:test`

Login succeeds immediately without any admin approval. The third column in the database was set to `True` at the point of insertion because the attacker injected a parameter the application developer intended to be internal.

***

## Why This Is Dangerous

The vulnerability does not require reverse engineering in a black-box context. In many cases, the field names can be inferred from:

- JavaScript source code or API responses revealing model attribute names
- Error messages or debug output exposing field names
- Comparing responses between different account types to deduce hidden boolean flags
- API documentation or OpenAPI specs that list all available fields including internal ones

In the Ruby on Rails example above, the attacker does not need to know the attribute is called `admin` in advance. Sending a broad parameter spray such as `role=admin`, `is_admin=1`, `confirmed=1`, `approved=true` across the registration request often hits valid field names by inference.

***

## Prevention

### Ruby on Rails: Strong Parameters

Replace the old `attr_accessible` approach with explicit whitelisting in the controller:

```ruby
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to @user
    else
      render 'new'
    end
  end

  private

  def user_params
    params.require(:user).permit(:username, :email)
  end
end
```

The `permit(:username, :email)` call produces a new hash containing only those two keys. Any additional parameters, including `admin`, `confirmed`, or `role`, are silently stripped before reaching the model.

### General Principles

- Never accept the full request body as a model directly. Always map explicitly to permitted fields
- Treat internal status fields like `confirmed`, `is_admin`, and `role` as server-set only. They should never appear in forms or be accepted from client input
- Implement integration tests that verify privileged fields cannot be set via the registration and update endpoints
- Use framework-level deny-by-default mass assignment so that every attribute must be explicitly opted in rather than opted out
