---
layout: post
title: Securing ASP.NET WebAPI using Client Certificates
tags : [webapi, ssl, certs, aspnet]
---
{% include JB/setup %}

There is no builtin support for client certificates in ASP.NET WebAPI. However it's easy to create a `DelegatingHandler` that intercepts all requests and checks the existance of a client certificate and it's value. This will only work if you host on IIS, because WebAPI as of today does not expose the client certificate in its object model.

I've created a NuGet package that will add the `DelegatingHandler` to your project and inject it on the `GlobalConfiguration` using `WebActivator`.

	Install-Package Auth10.AspNet.WebApi.ClientCert

**UPDATE**: adding #aspnet #webapi to your tweets is like sending the bat signal to [Henrik](https://twitter.com/frystyk) and [Glenn](https://twitter.com/gblock) from Microsoft. Good to know :). They pointed out that there is already [support for Client Certs](http://aspnetwebstack.codeplex.com/SourceControl/changeset/view/98d041ae352f#src%2fSystem.Net.Http.Formatting%2fHttpRequestMessageExtensions.cs) in a host-agnostic fashion but this is only available on the [nightly builds](http://blogs.msdn.com/b/henrikn/archive/2012/06/01/using-nightly-asp-net-web-stack-nuget-packages-with-vs-2012-rc.aspx) for now. There will be a `Request.GetClientCertificate` extension method that will get you that. I will update this NuGet to use that when WebAPI ships the RTM NuGets through the official channel.

## The devil is in the details

This looks easy, however the most challenging part is configuring IIS and add the certificates into the right place. Here are the instructions that will save you a couple of hours of research. These are also included in the README opened once you install the NuGet package.

1 Open IIS Manager and go to **SSL Settings** on your web site and check **Accept**. 

2 Enable **Anonymous Authentication**.

3 Open a command prompt and create a certificate that can be used for Client Authentication. These are the commands you can use to create a Certificate Authority and a certificate issued by that authority. If you have your own certificate issued by a trusted root authority this is not needed.

	makecert.exe -r -n "CN=My Personal CA" -pe -sv MyPersonalCA.pvk -a sha1 -len 2048 -b 01/21/2010 -e 01/21/2016 -cy authority MyPersonalCA.cer
	makecert.exe -iv MyPersonalCA.pvk -ic MyPersonalCA.cer -n "CN=John Doe" -pe -sv JohnDoe.pvk -a sha1 -len 2048 -b 01/21/2010 -e 01/21/2016 -sky exchange JohnDoe.cer -eku 1.3.6.1.5.5.7.3.2
	pvk2pfx.exe -pvk JohnDoe.pvk -spc JohnDoe.cer -pfx JohnDoe.pfx -po THE_PASSWORD_USED

4 Add MyPersonalCA.cer to Local Machine -> Truted Root Certificate Authorities. This is key, espcially while you are developing and want to try things. If you don't do it, `Request.ClientCertificate` won't be populated because the cert chain is untrusted.

## Source code

As a reference here is the code.

	public class X509CertificateHandler : DelegatingHandler
    {
        public Func<X509Certificate2, bool> ValidateCertificate { get; set; }

        public X509CertificateHandler(Func<X509Certificate2, bool> validateCertificate)
        {
            this.ValidateCertificate = validateCertificate;
        }

        protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
        {
            if (!HttpContext.Current.Request.ClientCertificate.IsPresent)
            {
                return Task.Factory.StartNew(() => request.CreateResponse(HttpStatusCode.Unauthorized));
            }

            var cert = new X509Certificate2(HttpContext.Current.Request.ClientCertificate.Certificate);
            if (!this.ValidateCertificate(cert))
            {
                return Task.Factory.StartNew(() => request.CreateResponse(HttpStatusCode.Unauthorized));
            }

            var identity = new GenericIdentity(cert.Subject, "ClientCertificate");
            var principal = new GenericPrincipal(identity, null);
            Thread.CurrentPrincipal = principal;
            HttpContext.Current.User = principal;

            return base.SendAsync(request, cancellationToken);
        }
    }

And here how you use it from your service.

    GlobalConfiguration.Configuration.MessageHandlers.Add(new X509CertificateHandler(cert => cert.Thumbprint.Equals("75c67d39fa6b......38b7fd40", System.StringComparison.OrdinalIgnoreCase)));

Finally, if you want to call a service using client certs in .NET. This is how you do it (this will also work with `WebClient` and `HttpClient` with some extension code using `MessageHandlers` on the client side).

    var client = WebRequest.Create(baseAddress + "api/hello") as HttpWebRequest;
    var cert = new X509Certificate2(File.ReadAllBytes("JohnDoe.pfx"), "PASSWORD");
    client.ClientCertificates.Add(cert);
    string response = new StreamReader(client.GetResponse().GetResponseStream()).ReadToEnd();
