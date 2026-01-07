# ID Upload Fix Documentation

## Problem Identified
The ID photo upload functionality is currently broken and not working. This document provides diagnostic steps and solutions to resolve the issue.

## Root Causes (Likely Issues)

### 1. **Storage Bucket Configuration**
- Storage bucket permissions may not be correctly set for authenticated users
- CORS policies might be blocking file uploads from the frontend
- Storage policies (RLS) might be denying write access

### 2. **Client-Side Upload Handler**
- The upload component might have errors in handling the file submission
- Missing error handling or validation before upload
- File size limitations not properly enforced

### 3. **API Endpoint Issues**
- The ID verification endpoint might be misconfigured
- Missing environment variables for Supabase
- Incorrect authentication headers

## Solution Steps

### Step 1: Verify Storage Bucket Setup
```sql
-- In Supabase SQL Editor, run:
SELECT * FROM storage.buckets WHERE name = 'id-documents';

-- Expected output: Row should exist with public = false
```

### Step 2: Check RLS Policies
```sql
-- Verify INSERT policy exists for authenticated users
SELECT definition FROM pg_policies 
WHERE tablename = 'objects' 
AND schemaname = 'storage'
AND qual LIKE '%authenticated%';
```

### Step 3: Fix CORS Settings in Supabase
1. Go to Supabase Dashboard
2. Navigate to **Storage** > **id-documents** (bucket)
3. Click **Settings** tab
4. Add CORS configuration:
```json
[
  {
    "origin": "*",
    "methods": ["GET", "POST", "PUT", "DELETE"],
    "headers": ["Content-Type", "authorization"],
    "maxAgeSeconds": 3600
  }
]
```

### Step 4: Update Frontend Upload Handler

Ensure your React component has proper error handling:

```typescript
const uploadIDPhoto = async (file: File) => {
  try {
    // Validate file
    if (!file) throw new Error('No file selected');
    if (file.size > 10 * 1024 * 1024) throw new Error('File too large (max 10MB)');
    if (!['image/jpeg', 'image/png', 'image/webp'].includes(file.type)) {
      throw new Error('Invalid file type. Use JPG, PNG, or WebP');
    }

    const supabase = createClient(supabaseUrl, supabaseAnonKey);
    const user = (await supabase.auth.getUser()).data.user;

    if (!user) throw new Error('User not authenticated');

    const fileName = `${user.id}/${Date.now()}-${file.name}`;

    const { data, error } = await supabase.storage
      .from('id-documents')
      .upload(fileName, file, {
        cacheControl: '3600',
        upsert: false
      });

    if (error) throw error;
    
    console.log('Upload successful:', data);
    return data;
  } catch (error) {
    console.error('Upload failed:', error);
    throw error;
  }
};
```

### Step 5: Enable Storage Debugging

Add logging to track the upload process:

```typescript
console.log('Starting upload...');
console.log('User ID:', userId);
console.log('File:', { name: file.name, size: file.size, type: file.type });
console.log('Bucket:', 'id-documents');
```

## Testing the Fix

1. **Verify RLS Policies**: Run the SQL queries above
2. **Test Upload**: Try uploading a small test image (<1MB)
3. **Check Bucket**: View uploaded files in Supabase Storage dashboard
4. **Verify Logs**: Check browser console for detailed error messages

## Common Error Codes

| Error | Cause | Solution |
|-------|-------|----------|
| 403 Forbidden | Storage permissions denied | Check RLS policies |
| 413 Payload Too Large | File exceeds size limit | Reduce file size |
| 415 Unsupported Media Type | Invalid file type | Use JPG/PNG/WebP |
| Network error | CORS blocked | Enable CORS in bucket settings |
| 401 Unauthorized | Not authenticated | Check auth session |

## Next Steps

1. Apply the fixes in order (steps 1-4)
2. Test the upload functionality
3. Monitor browser console and Supabase logs
4. If issues persist, check Supabase Dashboard > Logs > Edge Function invocations

## Support

For additional help, check:
- Supabase Documentation: https://supabase.com/docs/guides/storage
- GitHub Issues: Report with error logs and browser console output
