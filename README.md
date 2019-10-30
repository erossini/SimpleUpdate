# Simple Update for ASP.NET Core and Azure Storage
Simple update with drag &amp; drop in ASP.NET Core webapp to Azure Storage

To summarize what the code does, here’s a step-by-step explanation:

* Upload a file submitted in a web browser via HTTP POST.
* Convert the file to a byte array.
* Save the file data to Azure Blob Storage.

Let’s take a deeper look at the code.

1. The Index view under the Home controller displays one input field for selecting files for upload, and a submit button to complete the upload process.

Here’s a snippet:

```
<form enctype="multipart/form-data" method="post">
...
   <input multiple="multiple" name="files" type="file" />
...
   <input type="submit" value="Upload" />
...
</form>
```

2. Next, the Post method of the Home controller handles the uploaded file. It’s set up to loop through multiple uploaded files, and upload each file separately. Updated from the original blog post, there are now two options.

* Option A: convert to byte array before upload
* Option B: read directly from stream for blob upload

The UploadToBlob() method still takes in the filename. But now, it can take either take in the byte array or the stream of data from the uploaded file. If one of them is set to null, the other will be used. If both are null, the method will return false.

```
// NOTE: uncomment either OPTION A or OPTION B to use one approach over another

// OPTION A: convert to byte array before upload
//using (var ms = new MemoryStream())
//{
// formFile.CopyTo(ms);
// var fileBytes = ms.ToArray();
// uploadSuccess = await UploadToBlob(formFile.FileName, fileBytes, null);

//}

// OPTION B: read directly from stream for blob upload 
using (var stream = formFile.OpenReadStream())
{
 uploadSuccess = await UploadToBlob(formFile.FileName, null, stream);
}
```

You could change the code to upload them all to the same container, but that is up to you.

3. Finally, the `UploadToBlob()` method in the Home controller uploads the image data to a storage container in three stages: create a CloudBlobClient object to represent the storage account, create a container within that storage account and sets its permissions, upload the image data.

```
CloudBlobClient cloudBlobClient = storageAccount.CreateCloudBlobClient();

...
cloudBlobContainer = cloudBlobClient.GetContainerReference("uploadblob" + Guid.NewGuid().ToString());
await cloudBlobContainer.CreateAsync();

... 
BlobContainerPermissions permissions = new BlobContainerPermissions
{
   PublicAccess = BlobContainerPublicAccessType.Blob
};
await cloudBlobContainer.SetPermissionsAsync(permissions);

...
CloudBlockBlob cloudBlockBlob = cloudBlobContainer.GetBlockBlobReference(filename);

if (imageBuffer != null)
{
   // OPTION A: use imageBuffer (converted from memory stream)
   await cloudBlockBlob.UploadFromByteArrayAsync(imageBuffer, 0, imageBuffer.Length);
}
else if (stream != null)
{
   // OPTION B: pass in memory stream directly
   await cloudBlockBlob.UploadFromStreamAsync(stream);
} else
{
   return false;
}

return true;
```

For obvious security reasons, the connection string for the storage connection is kept in a config file that is not included in the Github repo. Instead there is a placeholder config file that can be renamed and updated to refer to any storage account that you own.
From the Readme file, the instructions are:

* Rename placeholder config file “appsettings.Development.txt” to “appsettings.Development.json”
* Replace placeholder string “_<REPLACE_CONN_STRING>_” with your Azure Storage Account connection string.

## Drag and drop

Another part of the code is for the drag and drop.

In the Index you find a div where you can drop your files. With `jQuery` I detect when you drop your files and then I call **UploadMyFiles** in the **HomeController**. 
