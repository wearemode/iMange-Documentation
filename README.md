# iMange Shopify Integration
This application handles the intergration between Shopify, Docebo and ReadyTech for iManage Learning Center. The Application is built using the [Laravel](https://laravel.com/) framework.

## Managing products in Shopify
To manage the integration between the different services we need to set two meta fields in the Shopify admin for the product variant's. The first is `docebo_id` which is the course id from Docebo. The second meta field is `readytech_id` which is the template id from ReadyTech which will enroll the user on the correct course. When these two metafields have been added a `App\Course` record will be created in the application database, this is detailed in the `App\Jobs\SyncShopifyProduct` job below.

## Shopify Webhook
When an order is placed on Shopify data will be posted to our application via a webhook. Which allows us to react to events rather than polling the API for changes.

When the webhook receives data the first step is to check the HMAC to verify the request is coming from Shopify. Once the the request has been verified we can then check the line items to see if a variant has been purchased which has been setup with Docebo and ReadyTech courses. This is done by checking the database to see if the `shopify_variant_id` exists in the `courses` table in the database.

If the query returns a `App\Course` model then we dispatch two jobs to enroll the student on the Docebo and ReadyTech courses. These jobs are `App\Jobs\EnrollUserOnDoceboCourse` and `App\Jobs\EnrollUserOnReadyTechCourse` which are detailed below.

## Queue Jobs
The applicationg uses [jobs](https://laravel.com/docs/7.x/queues), which are are dispatched to a queue to be run in the background, this is really useful for procceses which make external API requests as they are self contained and can be easily reattampted if anything should fail e.g. due to small network outages.

### App\Jobs\SyncAllShopifyProducts
This job is dispatched from a cron job every 5 minutes and queries all products from the iMange Shopify store and then dispatches a `App\Jobs\SyncShopifyProduct` job for each product that is returned.

### App\Jobs\SyncShopifyProduct
This job will load the metafields for the product variants, if both the `docebo_id` and `readytech_id` metafields are present then a new `App\Course` record is created in the database for each variant with meta field values used with the integration for the two API's.

### App\Jobs\EnrollUserOnDoceboCourse
This job handles the integration with Docebo, the first step is to create a new user using the following fields.

| Field         | Description            |
| ------------  | ---------------------- |
| userid        | Student's email        |
| email         | Student's email        |
| firstname     | Student's first name   |
| lastname      | Student's last name    |
| password      | Random 8 letter string |

The API will throw and error with the code 1002 if the user already exists, if this happens we catch the exception and find the user by their email address. As this is an existing record we won't know their password so this can't be emailed to them.

Now we have the `user_id` this can be used to enroll the user on the course using the Docebo API. We then also update the `App\Student` model to include the password if they are a new user on Docebo, otherwise this is set to `null`.

### App\Jobs\EnrollUserOnReadyTechCourse
This job handles the integration with ReadyTech. The first step is to check if the user's email address already exists on ReadyTech using the `get_users` function, this will return the user data or `false` if nothing is found. If no record is found then we create a new user record with the following information.

| Field         | Description            |
| ------------  | ---------------------- |
| username      | Student's email        |
| primaryEmail  | Student's last name    |
| firstName     | Student's first name   |
| lastName      | Student's last name    |

Then we need to generate a voucher code for the user to use to enroll on the correct course, this is done in two steps. The first is to create a new voucher block with the following properties and using the template id.

| Field            | Value |
| ---------------- | ----- |
| blockSize        | 1     |
| expirationPeriod | 30    |
| perVoucherCost   | 0.0   |

This will return the UUID for the voucher block, the final step is to get the contents of the voucher block and the first row will be the voucher code for the student. Which will be saved to the `App\User` record which can be emailed to the student.
