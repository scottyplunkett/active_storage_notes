# Active Storage:

Resources:

- <a href="https://weblog.rubyonrails.org/2017/11/27/Rails-5-2-Active-Storage-Redis-Cache-Store-HTTP2-Early-Hints-Credentials/">Rails-5-2-Active-Storage-Redis-Cache-Store-HTTP2-Early-Hints-Credentials/</a>
- <a href="https://evilmartians.com/chronicles/rails-5-2-active-storage-and-beyond">rails-5-2-active-storage-and-beyond
</a>

## Why does active storage exists?

- Pains of managing cloud storage providers, authentication, and data operations (an extremely common use case)
- Multiple packages for handling this existed in the form of Gems, many of which worked perfectly well, it was still a forced choice, forced configuration and external dependency in a system designed around the guiding principle of Convention over Configuration



##### If you want some examples of gem solutions here are the two most common:
###### - <a href="https://github.com/thoughtbot/paperclip">paperclip</a>
###### - <a href="https://github.com/carrierwaveuploader/carrierwave">carrierwave</a>

## Who created it?
<blockquote cite="https://weblog.rubyonrails.org/2017/11/27/Rails-5-2-Active-Storage-Redis-Cache-Store-HTTP2-Early-Hints-Credentials/">
  “Active Storage was extracted from Basecamp 3 by George Claghorn and yours truly. So not only is the framework already used in production, it was born from production. There’s that Extraction Design guarantee stamp alright!”
<b>- DHH</b>
</blockquote> 

## Hwo does it work?
##### Checkout the <a href="https://github.com/rails/rails/tree/master/activestorage">Acitve Storage Git</a>
#### A key difference to how Active Storage works compared to other attachment solutions in Rails is through the use  of built-in models (backed by Active Record): 
1. <a href="https://api.rubyonrails.org/classes/ActiveStorage/Blob.html">Active storage blobs</a>
  have data about the file.
2. <a href="https://api.rubyonrails.org/classes/ActiveStorage/Attachment.html">Active storage attachments</a> associate records with blobs.
#### This means existing application models do not need to be modified with additional columns to associate with files. Active Storage uses polymorphic associations via the Attachment join model, which then connects to the actual Blob.
  #### Blob models store attachment metadata (filename, content-type, etc.), and their identifier key in the storage service. **Blob models do not store the actual binary data.** They are intended to be immutable in spirit. One file, one blob. You can associate the same blob with multiple application models as well. And if you want to do transformations of a given Blob, the idea is that you'll simply create a new one, rather than attempt to mutate the existing one (though of course you can delete the previous version later if you don't need it).

#### I'm confused where is my file? 
##### Not in the db. It's on a cloud service... that's kind of the whole point. However, during development, so long as your `config/storage.yml` is set up to use the disk service locally, you're file will be stored in the `./storage` directory or `.tmp/storage/` for testing. (An example `config/storage.yml` file can be found in the next section)


## How is it configured in a rails project?

---
#### In `config/storage.yml` you configure vaious storage providers. 


```yaml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
 
test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>
 
amazon:
  service: S3
  access_key_id: ENV['AWS_ACCESS_KEY']
  secret_access_key: ENV['AWS_SECRET_ACCESS_KEY']
  bucket: 'some-bucket-name'
  region: 'us-east-1'

google:
  service: GCS
  credentials: <%= Rails.root.join("path/to/keyfile.json") %>
  project: "google-project"
  bucket: "gcs-bucket"
```

##### NOTE: You'll probably also need to install a gem for the providers you're using.
###### <a href="https://github.com/aws/aws-sdk-ruby">aws gem</a>
###### <a href="https://github.com/GoogleCloudPlatform/google-cloud-ruby/tree/master/google-cloud-storage">google cloud storage gem</a>
<hr/>

---
#### These tables in `db/schema.rb` are set up for you in new projects. Run `rails active_storage:install` to copy over their migrations.

```ruby
  create_table "active_storage_attachments", force: :cascade do |t|
    t.string "name", null: false
    t.string "record_type", null: false
    t.bigint "record_id", null: false
    t.bigint "blob_id", null: false
    t.datetime "created_at", null: false
    t.index ["blob_id"], name: "index_active_storage_attachments_on_blob_id"
    t.index ["record_type", "record_id", "name", "blob_id"], name: "index_active_storage_attachments_uniqueness", unique: true
  end

  create_table "active_storage_blobs", force: :cascade do |t|
    t.string "key", null: false
    t.string "filename", null: false
    t.string "content_type"
    t.text "metadata"
    t.bigint "byte_size", null: false
    t.string "checksum", null: false
    t.datetime "created_at", null: false
    t.index ["key"], name: "index_active_storage_blobs_on_key", unique: true
  end

```

---
#### This in your model file (ex. uses `./models/lesson.rb` where Model is Lesson and name of the attachment is video.

```ruby
class Lesson < ApplicationRecord
  # Associates an attachment and a blob. When the user is destroyed they are
  # purged by default (models destroyed, and resource files deleted).
  has_one_attached :video
end
```

#### or for multiple:
```ruby
class Lesson < ApplicationRecord
  # Associates an attachment and a blob. When the user is destroyed they are
  # purged by default (models destroyed, and resource files deleted).
  has_many_attached :video
end
```

---
#### This in some controller action for a one-to-one relationship:
`@lesson.video.attach(lesson_params[:video])`

#### Or in the parent entity's controller (ex. uses `./controller/lessons_controller.rb`):
```ruby
class LessonsController < ApplicationController
  def create
    @lesson = Lesson.create!(lesson_params)
    redirect_to @lesson
  end

  def update
    @lesson = Lesson.find(lesson_params[:lesson])
    @lesson.update!(lesson_params)
    redirect_to @lesson
  end

  private 

  def lesson_params
    params.require(:lesson).permit(:video, :title)
  end
end
```

#### Multiple:
`@lesson.videos.attach(params[:videos])`

#### Or multiple in the parent entity's controller (ex. uses `./controller/lessons_controller.rb`):
```ruby
class LessonsController < ApplicationController
  def create
    lesson = Lesson.create!(lesson_params)
    redirect_to lesson
  end
 
  private
    def lesson_params
      params.require(:lesson).permit(:title, videos: [])
    end
end
```

---
#### Using with API-only apps
  - Wrap in form-data object 

```javascript
// example submit function
submit = (lesson, videos) => {
  const formData = new FormData();
  videos.forEach(video => { formData.append('payload[]', video) });
  formData.append('files', true);
  LessonsAPI(lesson).post(formData)
    .then(videos => {
      videos.forEach(video => someSuccessFunction(video))
      this.close()})
    .catch(error => someFailFunction(error))
}
```


  - Watch out for JSON.stringify in POST request
  - Watch out for POST request with the content-type header set

```javascript
// related example post function in some request wrapper
post: (url, data, options = { headers: {}}) => {
    let body;

    if (data instanceof FormData && data.has("files")) {
      body = data
    } else {
      options.headers["Content-Type"] = "application/json"
      body = JSON.stringify(data);
    }

    return fetch(url, {
      method: "POST",
      body: body,
      ...options
    })
  }
```


### What if I want to upload directly from the client?
<a href="https://edgeguides.rubyonrails.org/active_storage_overview.html#direct-uploads"> Direct Uploads </a>
