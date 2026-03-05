# StreamingApp: End-to-End User Guide

Welcome to the StreamingApp! This guide will walk you through the entire lifecycle of using the platform—from creating your first account, to securing administrative privileges, uploading video content, and streaming it.

---

## 🛠️ Prerequisites
- The application must be fully deployed and running on your Kubernetes cluster.
- You should have `kubectl` installed on your terminal to interact with the database.

---

## 👤 Step 1: Initial Registration & Login

1. **Access the Website**
   Open your preferred web browser and navigate to the application's URL. If you are running locally, go to:
   `http://localhost` (or the CloudFront domain if hosted on AWS).
   
2. **Create an Account**
   - Click the **"Sign Up"** or **"Register"** button on the homepage.
   - Fill in your details (e.g., `user@example.com`, `Password123`).
   - Submit the form to create your account. By default, every new account is created as a **Standard User**.

3. **Log In**
   - You should be automatically redirected to the login screen.
   - Enter the email and password you just registered.
   - You are now authenticated! You should see your user profile in the top navigation bar.

---

## ⚡ Step 2: Elevating Your Account to Administrator

As a standard user, you can watch videos, but you *cannot* access the upload dashboard. The `/admin` portal is securely protected by React's client-side routing and the Node.js backend to prevent unauthorized video uploads.

To unlock this feature, you must manually promote your account inside the Kubernetes database.

1. **Open your Terminal (Mac/Linux/Windows)**
2. **Connect to the Database Pod**
   Execute the following command to automatically find your running MongoDB pod and open an interactive shell (`mongosh`) inside the `streamingapp` database:
   ```bash
   kubectl exec -it $(kubectl get pod -l component=database -o jsonpath='{.items[0].metadata.name}') -- mongosh streamingapp
   ```
3. **Promote the User Role**
   Once you see the MongoDB prompt (`test>`), run the following javascript command to grant yourself admin access. ***Make sure to replace the email inside the quotes with the exact email you signed up with:***
   ```javascript
   db.users.updateOne({ email: "user@example.com" }, { $set: { role: "admin" } });
   ```
   *You should see an output confirming the `matchedCount: 1` and `modifiedCount: 1`.*
4. **Exit the Database**
   Type `exit` and press enter to return to your normal computer terminal.

---

## 📤 Step 3: Accessing the Admin Dashboard

1. **Refresh Your Session**
   Because your browser technically still has a "Standard User" token saved, you need to refresh it.
   - Click **Logout** on the application's top navigation bar.
   - **Log back in** with your email and password. Your new token will now successfully state that you are an `admin`.
2. **Open the Dashboard**
   - Navigate to `http://localhost/admin` in your browser. (Or click the Admin Dashboard link if available in the UI).
   - The private administrator upload dashboard will now appear correctly instead of redirecting you back to the home page!

---

## 🎬 Step 4: Uploading a Video & Thumbnail

From inside the Admin Dashboard:

1. **Upload the Media Files**
   - Locate the Video Upload section.
   - Select the **Video File** (`.mp4`, `.mov`, etc.) you wish to stream from your personal computer.
   - Select a high-quality **Thumbnail Image** (`.jpg`, `.png`) to be displayed as the preview card.
2. **Fill in the Metadata**
   - Provide a compelling **Title** and **Description** for the video.
   - Select any relevant categories or tags provided by the menu.
3. **Publish**
   - Click the "Upload" or "Submit" button. 
   - Wait for the progress bar to complete. The backend `admin-service` handles processing the upload securely into your cloud/local storage.

---

## 🍿 Step 5: Streaming Your Content

1. **Return to the Homepage**
   Navigate back to `http://localhost/browse` or click the website's logo.
2. **Find Your Video**
   You should now see the thumbnail you uploaded appearing dynamically on the main grid of the application!
3. **Watch the Stream**
   - Click on the thumbnail.
   - The application will route you to the dynamic video player page. 
   - Press **Play**. The frontend will connect to the `streaming-service`, requesting the video chunks and caching them into the HTML5 video player for a buttery smooth streaming experience.

---

## 🚪 Step 6: Logging Out

When you are finished managing content or watching streams:
1. Click your profile name / avatar in the top right corner.
2. Select **Logout** from the dropdown menu.
3. Your secure JWT token will be destroyed, closing your session and returning you to the login screen.

---
*Happy Streaming!*
