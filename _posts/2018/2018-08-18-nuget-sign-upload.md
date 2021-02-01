---
layout: post
title: Signing a nupkg and Uploading to Nuget.org
tags:
- nuget
- nupkg
- public-key
---

[A few days ago]({% post_url /2018/2018-08-15-nupkg-with-native %}) I detailed how to create a nupkg containing a native library.  Next we'll sign and upload it to [nuget.org](https://www.nuget.org/).

Nuget.org also has [an excellect post](https://blog.nuget.org/20180522/Introducing-signed-package-submissions.html) on the topic.

Motivations:
- Make our own libraries more accessible to others.  Previously we provided instructions on cloning the repo and bringing it into an existing project- using nuget would be easier.
- Simplify our codebase.  Our main .sln has 55 projects; perhaps 10 of which are third party libs that we made minor changes to but otherwise rarely touch.

## Organization

We want all our packages to be owned by the [Subor organization](https://www.nuget.org/profiles/subor) so everything won't be tied to a single person.

After creating an organization (profile __> Manage Organizations > Add new__), you can click the pencil icon (![]({{ "/assets/nuget_edit_icon.png" | absolute_url }})) to access the organization's settings page and configure various things.

## Certificate Creation and Registration

Assuming you have a _PKCS #12_ file (with a .pfx extension) for signing, you need to export a .cer to register with nuget.org.  On Windows, double-clicking the pfx should launch the Certificate Import Wizard, or you can run `certmgr.exe` and click __Import...__.

Once imported, select the key and click __Export...__:  
![]({{ "/assets/certmgr_export.png" | absolute_url }})

When prompted select __No, do not export the private key__:  
![]({{ "/assets/certmgr_export_no_private.png" | absolute_url }})

And __DER encoded X.509__:  
![]({{ "/assets/certmgr_export_format.png" | absolute_url }})

Then click through to create the .cer.  Now, go to the organization's settings page on nuget.org and pick __Certificates > Register new__.  Select the .cer you just created.

## Nuget.org API Key

While you're visiting nuget.org it's worth creating an API key __profile > API Keys > +Create__:  
![]({{ "/assets/nuget-org_apikey_create.png" | absolute_url }})

For __Package Owner__ pick the organization.  Uploaded packages will belong to the organization rather than your individual account.

For __Select Scopes__ I've got __Push only new package versions__ because I'm planning to use this on our build machine and it really has no reason to create new packages.

Click __Create__ then __Copy__ to save the API key to your clipboard.  You cannot get this key again later; if you forget it you just have to __Regenerate__ it (invalidating the old one):  
![]({{ "/assets/nuget-org_apikey_regenerate.png" | absolute_url }})

## Signing

I've got `Subor.NNanomsg.NETStandard.0.5.2.nupkg` from the [other day]({% post_url /2018/2018-08-15-nupkg-with-native %}) and our private key (the pfx).  To sign the package:
```
nuget.exe sign Subor.NNanomsg.NETStandard.0.5.2.nupkg -Timestamper http://sha256timestamp.ws.symantec.com/sha256/timestamp -CertificatePath path_to_private_key.pfx
```

Output should be similar to:
```
Please provide password for: path_to_private_key.pfx
Password: ********************************
 
 
Signing package(s) with certificate:
  Subject Name: CN=???????????????????, O=???????????????????, L=??, C=CN
  SHA1 hash: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
  SHA256 hash: BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
  Issued by: CN=DDDDDDDDDDDDDDD SHA256 Code Signing CA, OU=EEEEEEEEEEEEEEEEE, O=FFFFFFFFFFFFFFFFFF, C=US
  Valid from: 12/34/56 00:00:00 to 12/34/56 00:00:00
 
Timestamping package(s) with:
http://sha256timestamp.ws.symantec.com/sha256/timestamp
Package(s) signed successfully.
```

After signing the size of the nupkg file should increase slightly (in this case 10KB).

If you get output that ends with `Key does not exist.`, make sure the .pfx (the private key) follows `-CertificatePath` and not the .cer (public key).

You can verify the package with:
```
nuget.exe verify Subor.NNanomsg.NETStandard.0.5.2.nupkg -All
```
And there should be a bunch of similar output that ends with `Successfully verified package 'Subor.NNanomsg.NETStandard.0.5.2'.`.

## Uploading

The first time I uploaded the package I used nuget.org's web interface: profile __> Manage Packages > +Add new__.

Thereafter I can use the "update package versions"-only API key I created to push updates:
```
nuget.exe push Subor.NNanomsg.NETStandard.0.5.2.nupkg -Source "https://www.nuget.org" -ApiKey abcdef01234567890abcdef01234567890
```
If everything is working:
```
Pushing Subor.NNanomsg.NETStandard.0.5.2.nupkg to the NuGet gallery (https://www.nuget.org)...
  PUT https://www.nuget.org/api/v2/package/
  Created https://www.nuget.org/api/v2/package/ 3158ms
Your package was pushed.
```

The package is validated before becoming available via nuget.org.  This seems to take around 5 minutes for a small package like this, although I'd imagine it might take longer for large packages or during peak usage.  You can keep an eye on its status via the package's page: [https://www.nuget.org/packages/Subor.NNanomsg.NETStandard/0.5.2](https://www.nuget.org/packages/Subor.NNanomsg.NETStandard/0.5.2).

## The Circle is Now Complete

Back in Visual Studio, you can now:
1. Right-click a project __> Manage NuGet Packages... > Browse__.
1. Make sure __Package source__ is `nuget.org` (in case you changed it during package development).  And search for `Subor.NNanomsg.NETStandard`:  
![]({{ "/assets/vs_nuget_browse.png" | absolute_url }})

1. __Install__ it and build the project (don't forget to [set __Platform target__]({% post_url /2018/2018-08-15-nupkg-with-native %}#consuming-nupkg)!).
1. Browse to the output folder and `nanomsg.dll` should be there.

Pretty slick.