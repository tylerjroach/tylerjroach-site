+++
author = "Tyler Roach"
categories = ["android", "java"]
date = "2016-09-01"
description = "A clean approach to managing runtime permissions"
title = "Reduce Runtime Permission Clutter (Headless Dialog Fragments!)"
featured = ""
featuredpath = ""
featuredalt = ""
type = "post"
+++

I know I’m late to the party implementing Android runtime permissions in the
[Stream](https://play.google.com/store/apps/details?id=com.sparc.stream) app. We
had our reasons for this, but now that Nougat has been released, its time we
upgrade our targetSdkVersion to 24.

I’m not a huge fan of the design for runtime permissions. It can quickly clutter
your activity/fragment code as you have to handle multiple grant scenarios with
proper messaging to the user. In order to combat this, I decided to contain
nearly all aspects of permission management inside headless dialog fragments.

To show how this works, I’ve created an example app that demonstrates how to
request all necessary permissions to record videos (camera, mic, and storage).
The full source code can be found here for reference:
[https://github.com/tylerjroach/RuntimePermissionsExample](https://github.com/tylerjroach/RuntimePermissionsExample).
Lets go ahead and look at the entire CameraPermissionsDialogFragment source.

```
public class CameraPermissionsDialogFragment extends DialogFragment{
  private final int PERMISSION_REQUEST_CODE = 11;

  private Context context;
  private CameraPermissionsGrantedCallback listener;

  private boolean shouldResolve;
  private boolean shouldRetry;
  private boolean externalGrantNeeded;

  public static CameraPermissionsDialogFragment newInstance() {
    return new CameraPermissionsDialogFragment();
  }

  public CameraPermissionsDialogFragment() {}

  @Override public void onAttach(Context context) {
    super.onAttach(context);
    this.context = context;
    if (context instanceof CameraPermissionsGrantedCallback) {
      listener = (CameraPermissionsGrantedCallback) context;
    }
  }

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setStyle(STYLE_NO_TITLE, R.style.PermissionsDialogFragmentStyle);
    setCancelable(false);
    requestNecessaryPermissions();
  }

  @Override public void onResume() {
    super.onResume();
    if (shouldResolve) {
      if (externalGrantNeeded) {
        showAppSettingsDialog();
      } else if(shouldRetry) {
        showRetryDialog();
      } else {
        //permissions have been accepted
        if (listener != null) {
          listener.navigateToCaptureFragment();
          dismiss();
        }
      }
    }
  }

  @Override public void onDetach() {
    super.onDetach();
    context = null;
    listener = null;
  }

  @Override public void onRequestPermissionsResult(int requestCode,
      String permissions[], int[] grantResults) {
    shouldResolve = true;
    shouldRetry = false;

    for (int i=0; i<permissions.length; i++) {
      String permission = permissions[i];
      int grantResult = grantResults[i];

      if (!shouldShowRequestPermissionRationale(permission) && grantResult != PackageManager.PERMISSION_GRANTED) {
        externalGrantNeeded = true;
        return;
      } else if (grantResult != PackageManager.PERMISSION_GRANTED) {
        shouldRetry = true;
        return;
      }
    }
  }

  private void requestNecessaryPermissions() {
    requestPermissions(new String[] {
        Manifest.permission.CAMERA,
        Manifest.permission.RECORD_AUDIO,
        Manifest.permission.WRITE_EXTERNAL_STORAGE}, PERMISSION_REQUEST_CODE);
  }

  private void showAppSettingsDialog() {
    new AlertDialog.Builder(context)
        .setTitle("Permissions Required")
        .setMessage("In order to record videos, access to the camera, microphone, and storage is needed. Please enable these permissions from the app settings.")
        .setPositiveButton("App Settings", new DialogInterface.OnClickListener() {
          @Override public void onClick(DialogInterface dialogInterface, int i) {
            Intent intent = new Intent();
            intent.setAction(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
            Uri uri = Uri.fromParts("package", context.getApplicationContext().getPackageName(), null);
            intent.setData(uri);
            context.startActivity(intent);
            dismiss();
          }
        })
        .setNegativeButton("Cancel", new DialogInterface.OnClickListener() {
          @Override public void onClick(DialogInterface dialogInterface, int i) {
            dismiss();
          }
        }).create().show();
  }

  private void showRetryDialog() {
    new AlertDialog.Builder(context)
        .setTitle("Permissions Declined")
        .setMessage("In order to record videos, the app needs access to the camera, microphone, and storage.")
        .setPositiveButton("Retry", new DialogInterface.OnClickListener() {
          @Override public void onClick(DialogInterface dialogInterface, int i) {
            requestNecessaryPermissions();
          }
        })
        .setNegativeButton("Cancel", new DialogInterface.OnClickListener() {
          @Override public void onClick(DialogInterface dialogInterface, int i) {
            dismiss();
          }
        }).create().show();
  }

  public interface CameraPermissionsGrantedCallback {
    void navigateToCaptureFragment();
  }
}
```

In onCreate(), we call requestNecessaryPermissions(). This prompts the user to
deny/allow each requested permission, and the results are passed to
onRequestPermissionResult(). In this example, we are checking for 3 possible
scenarios:

1.  All permissions accepted
1.  One or more permissions denied
1.  One or more permissions denied and “Don’t ask again” checked

Regardless of the outcome, you will want to be careful in what you update in
onRequestPermissionResult(). The results are passed down to this method while
the app is in a paused state. As a result, certain actions can cause a crash
(Ex. IllegalStateException from a FragmentTransaction commit). To work around
this, I set boolean flags on the action to take, and built onResume() to handle
the action once the app is back into the foreground. This will guarantee that
fragment transactions will be safe to run, and the ui will be safe to update.

Now, lets go back to our activity that is requesting the runtime permissions.
You can see that the activity is only responsible for checking if any
permissions are needed before opening the camera. You can see how the activity
code could quickly become cluttered if we didn’t separate the permission
requests into our DialogFragment.

```
@Override protected void onResume() {
  super.onResume();
  if (isPermissionGranted()) {
    status.setText("All permissions granted. Ready to open Camera");
  } else {
    status.setText("Permissions Needed");
  }
}

@Override public void navigateToCaptureFragment() {
  if (isPermissionGranted()) {
    Toast.makeText(this, "Opening Camera!", Toast.LENGTH_LONG).show();
  } else {
    CameraPermissionsDialogFragment.newInstance().show(getSupportFragmentManager(), CameraPermissionsDialogFragment.class.getName());
  }
}

private boolean isPermissionGranted() {
  return ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) == PackageManager.PERMISSION_GRANTED &&
      ContextCompat.checkSelfPermission(this, Manifest.permission.RECORD_AUDIO) == PackageManager.PERMISSION_GRANTED &&
      ContextCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE) == PackageManager.PERMISSION_GRANTED;
}
```


Side notes:

* I used a DialogFragment, however, a standard Headless Fragment could have worked
as well. I chose to use a dialog fragment because setCancelable(false) can
easily prevent all outside touches, as well as the simple creation/destruction
with show() and dismiss().
* By default, a DialogFragment dims the screen background content. To remove this,
simply use this style on the dialog.

```
<style name="PermissionsDialogFragmentStyle" parent="Base.Theme.AppCompat.Dialog">
   <item name="android:backgroundDimEnabled">false</item>
</style>
```

I look forward to reading any comments or questions you may have!

