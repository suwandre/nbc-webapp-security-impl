# NBC Webapp Security Implementation
Frontend repo: https://github.com/NotBoringCompany/nbc-webapp
Backend repo: https://github.com/NotBoringCompany/nbc-webapp-api

## 1. Threat Model
Below is a threat model for our web app:
![image](https://user-images.githubusercontent.com/60882255/235114351-64b90af6-5c0c-49b4-82f0-7026b619e224.png)
More details regarding the infrastructure will be explained in the upcoming points.

What we plan to add in our web app are the following (that will impact the threat model in the future):
- Email and password signing up and authentication using OAuth

## 2. Authentication and authorization (webapp)
The only way users can connect to the web app is via Metamask. I've used Moralis' SDK to help me simplify this process but without mitigating security. When a user logs in with Metamask, a signature message from the backend is generated, and when the user accepts the message, it is then encrypted and reverted back as a unique session token which is used to identify and authenticate the user in our web app. Session tokens expire in 3 days. 

As we haven't finished building the email-password signup method, there are currently no known risks of authentication/authorization breaches from our side for the web app.

## 3. Authentication and authorization (backend, DB, blockchain)
All CRUD operations that instantiates MongoDB via our backend have middlewares to ensure that only authorized users are allowed to access these endpoints. For instance: we are implementing soft staking for our NFTs, which means users won't have to spend gas and interact with our contract to stake (known as hard staking). However, this means that we need to make sure users can only stake their own NFTs, that they can't add their NFTs multiple times on the same staking pool multiple times to get more rewards or add someone else's NFTs on their behalf. 

Checks have been made to make sure that these users are themselves (by checking via their session token), and all other checks that don't have security implementations are functions that don't impose any risks to anyone and rather even potentially benefitting the 'victim' instead, for example when unstaking the NFTs and claiming the rewards (as by default, unstaking is NOT allowed when the staking pool has started, and claiming rewards will claim it directly to the staker that owns the subpool).

As smart contracts are used as the main source of truth for our current operations (staking), all authorization methods have been properly implemented in the smart contracts via modifiers and reverts to ensure that users cannot, for example, withdraw more tokens than allowed from the admin wallet, for instance when earning the token rewards.

## 4. NoSQL injection prevention
Using the MongoDB driver (Golang), we've applied some best practices from some sources such as using parameterized queries to ensure that user input is properly escaped and validated before being used in a query. Across the backend, I've implemented the usage of `bson.M{}` and `bson.D{}` types throughout all CRUD operations. 

More implementations such as using `bsoncore.ValidateString` to validate string input will be implemented in the near future.

## 5. Web App Security
### a. Vercel as a deployment platform
We are using Vercel as our frontend platform to deploy our web app, which automatically provisions and configures a TLS certificate from Let's Encrypt, a certificate authority that provides encryption for all traffic to and from our web app. This helps protect against eavesdropping, tampering and other sorts of security threats when transmitting data. 

Vercel also includes DDoS protection, firewall rules and access controls.
### b. Security Headers
The following can be found in the (Next config file)[https://github.com/NotBoringCompany/nbc-webapp/blob/main/next.config.js] of the web app repo.
```
async headers() {
		return [
			{
				source: "/(.*)",
				headers: createSecureHeaders({
					contentSecurityPolicy: {
						directives: {
							frameAncestors: [
								"self",
							],
							styleSrc: ["'self'", "'unsafe-inline'"],
							imgSrc: [
								"'self'",
                						"realmhunter-kos.fra1.cdn.digitaloceanspaces.com",
							],
							baseUri: "self",
						},
					},
					forceHTTPSRedirect: [
						true,
						{ maxAge: 63072000, includeSubDomains: true },
					],
					noopen: "noopen",
					nosniff: "nosniff",
					xssProtection: "sanitize",
					referrerPolicy: "strict-origin-when-cross-origin",
				}),
			},
		];
	},
  ```
This includes:
- CSP (Content Security Policy), which protects against certain types of attacks, such as cross-site scripting (XSS) and data injection attacks. A relatively strict CSP has been employed in our case: `frame-ancestors` are currently only self, but will soon be added with only domains that are trusted and to be used. `styleSrc` is set to `["'self'"]` which specifies that only styles from our own domain. `unsafe-inline` here poses a threat as it allows inline styles to be applied to our page considering improper input validation and sanitization, but we are planning to switch it to include a randomly generated (per each page load) `nonce` instead. `imgSrc` is also only allowed from the domains specified (which are the only ones we need). `baseUri` is self, which means that any relative URL will be interpreted as relative to our web app's own domain. It also prevents clickjacking attacks from other website from embedding our web app within an iframe, since the base URL won't match with the embedding website's domain.

- HSTS (HTTP Strict Transport Security), which force redirects any users who connects to our web app using http:// instead of https://. This helps prevent man-in-the-middle (MITM) attacks, season hijacking and improve security issues such as data breaches and identity theft. This can be seen in `forceHTTPSRedirect: [
						true,
						{ maxAge: 63072000, includeSubDomains: true },
					],`
- XSS (cross-site scripting) protection, which stops pages from loading when they detect reflected XSS attacks. This is used as we currently also still use `unsafe-inline`, but furthermore it does help provide protection to users that use older web browsers that still don't support CSP.

- Referrer policy control, which we set to `"strict-origin-when-cross-origin"`. When using this policy, the full URL of the referring page will only be sent in the `Referrer` header when the user navigates from one page on our web app to another page on our web app (i.e same origin requests). When a user navigates from a page on our web app to a page on another website (cross origin), only the origin of the referring page will be included in the `Referrer` header, which protects the user's privacy by preventing other websites from seeing the full URL of the referring page. This also prevents certain types of attacks such as cross-site request forgery (CSRF) attacks, which can be used to impersonate the user and perform actions on their behalf.

## 6. DDoS protection
Although we can't 100% prevent DDoS attacks and more implementations can be added, we've implemented the following for now:
- Rate limiting backend via Redis, which limits the amount of traffic per user.
- 24/7 monitoring, traffic filtering, Anycast DNS and automatic scaling, which are all provided by Vercel.
- Reducing request payload from the frontend to the backend, which can sometimes log up memory and CPU usage of the backend provider.
- CDN (content delivery network) usage for our media (in DigitalOcean Spaces), which caches the content we've uploaded and reduces bandwidth by up to 80%.

## 7. Password management
As we are currently building our email-password signup method, we are planning to use Mongo Passport and bcrypt to auto-generate a salt and encrypt each user's passwords when storing it in our database.

For password resets, we will use a secure token when being embedded in the password URL. This needs to be single use and randomly generated using `crypto.js` or similar crypto libraries that can generate a strong and cryptographically safe token and we will be sure to let it expire immediately after an appropriate period.

## 8. Captcha usage
Although this hasn't been implemented yet, we are planning to add Google's re-Captcha tool to our site to prevent spam and/or abuse (which can also help mitigate DDoS attacks).

## 9. Testing and functionality
We have created (and will continue to create) tests and sample instances (such as sample functions, sample collections in the database etc.) to ensure that each function that is required functions properly and no leaks, error potentials and unnnecessary error panics occur. Although not implemented yet, we are going to use Github Actions as a CI/CD tool to ensure that each push into the repository is clean of simple and known issues.
