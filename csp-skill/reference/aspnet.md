# CSP — ASP.NET WebForms / VB.NET (and C#)

Applies to: ASP.NET WebForms (.aspx), IIS, VB.NET or C# code-behind.
Also relevant for ASP.NET MVC with Razor views.

---

## IHttpModule (recommended approach)

Implement as an `IHttpModule` so the nonce is available to every page
without modifying individual code-behind files.

**Location**: `App_Code/CspModule.vb` (or `CspModule.cs`)

### VB.NET

```vb
Imports System.Web

Public Class CspModule
    Implements IHttpModule

    Public Sub Init(context As HttpApplication) Implements IHttpModule.Init
        AddHandler context.PostAcquireRequestState, AddressOf OnPostAcquireRequestState
        AddHandler context.EndRequest, AddressOf OnEndRequest
    End Sub

    Private Sub OnPostAcquireRequestState(sender As Object, e As EventArgs)
        Dim app As HttpApplication = CType(sender, HttpApplication)

        ' Generate nonce — 128 bits of cryptographic randomness
        Dim bytes(15) As Byte
        Using rng As New System.Security.Cryptography.RNGCryptoServiceProvider()
            rng.GetBytes(bytes)
        End Using
        Dim nonce As String = Convert.ToBase64String(bytes)

        ' Store nonce for use in master pages and controls
        app.Context.Items("CspNonce") = nonce
    End Sub

    Private Sub OnEndRequest(sender As Object, e As EventArgs)
        Dim app As HttpApplication = CType(sender, HttpApplication)
        Dim response As HttpResponse = app.Response
        Dim nonce As String = CStr(app.Context.Items("CspNonce") ?? "")

        ' Skip non-HTML responses
        If Not response.ContentType.StartsWith("text/html") Then Return

        ' Read config — use Web.config appSettings or your config system
        Dim enabled As Boolean = CBool(ConfigurationManager.AppSettings("CSP_ENABLED") ?? "true")
        Dim enforce As Boolean = CBool(ConfigurationManager.AppSettings("CSP_ENFORCE") ?? "false")

        If Not enabled Then Return

        Dim policy As String = BuildPolicy(nonce, enforce)
        Dim headerName As String = If(enforce, "Content-Security-Policy", "Content-Security-Policy-Report-Only")

        response.Headers.Set(headerName, policy)
    End Sub

    Private Function BuildPolicy(nonce As String, enforce As Boolean) As String
        If Not enforce Then
            Return ReportOnlyPolicy(nonce)
        End If
        Return EnforcementPolicy(nonce)
    End Function

    Private Function ReportOnlyPolicy(nonce As String) As String
        Return String.Join("; ", New String() {
            "default-src 'self'",
            $"script-src 'self' 'nonce-{nonce}' 'strict-dynamic' 'unsafe-inline'",
            "style-src 'self' 'unsafe-inline'",
            "font-src 'self' data:",
            "img-src 'self' data:",
            "connect-src 'self'",
            "object-src 'none'",
            "base-uri 'self'",
            "form-action 'self'",
            "report-uri /csp-report"
        })
    End Function

    Private Function EnforcementPolicy(nonce As String) As String
        Return String.Join("; ", New String() {
            "default-src 'self'",
            $"script-src 'nonce-{nonce}' 'strict-dynamic'",
            $"style-src-elem 'self' 'nonce-{nonce}'",
            "style-src-attr 'unsafe-inline'",
            "font-src 'self' data:",
            "img-src 'self' data:",
            "connect-src 'self'",
            "frame-ancestors 'none'",
            "object-src 'none'",
            "base-uri 'self'",
            "form-action 'self'",
            "upgrade-insecure-requests",
            "report-uri /csp-report"
        })
    End Function

    Public Sub Dispose() Implements IHttpModule.Dispose
    End Sub
End Class
```

## Register in Web.config

```xml
<system.webServer>
  <modules>
    <add name="CspModule" type="CspModule" />
  </modules>
</system.webServer>

<appSettings>
  <add key="CSP_ENABLED" value="true" />
  <add key="CSP_ENFORCE" value="false" />
</appSettings>
```

## Using the nonce in Master Pages (.master)

```aspx
<%-- In your Master Page code-behind or inline --%>
<%
    Dim nonce As String = CStr(Context.Items("CspNonce") ?? "")
%>

<%-- Apply to ScriptManager (WebForms) --%>
<asp:ScriptManager ID="ScriptManager1" runat="server" />

<%-- Apply to inline scripts --%>
<script nonce="<%= nonce %>" src="js/app.js"></script>

<%-- For unavoidable inline blocks --%>
<script nonce="<%= nonce %>">
    var config = { locale: '<%= Thread.CurrentThread.CurrentCulture.Name %>' };
</script>
```

## CSP report endpoint

Add a new `.ashx` Generic Handler:

**Location**: `csp-report.ashx`

```vb
<%@ WebHandler Language="VB" Class="CspReport" %>

Imports System.Web
Imports System.IO

Public Class CspReport
    Implements IHttpHandler

    Public Sub ProcessRequest(context As HttpContext) Implements IHttpHandler.ProcessRequest
        Dim body As String
        Using reader As New StreamReader(context.Request.InputStream)
            body = reader.ReadToEnd()
        End Using

        ' Log to file — adjust path as needed
        Dim logPath As String = context.Server.MapPath("~/App_Data/csp-violations.log")
        Dim entry As String = String.Format(
            "[{0}] body={1}{2}",
            DateTime.UtcNow.ToString("o"),
            body.Replace(Environment.NewLine, " "),
            Environment.NewLine
        )
        File.AppendAllText(logPath, entry)

        context.Response.StatusCode = 204
        context.Response.End()
    End Sub

    Public ReadOnly Property IsReusable As Boolean Implements IHttpHandler.IsReusable
        Get
            Return False
        End Get
    End Property
End Class
```

Update `report-uri` in the policy to `/csp-report.ashx`.

## WebForms-specific gotchas

| Issue | Cause | Fix |
|---|---|---|
| `__doPostBack` blocked | WebForms registers inline `onclick` for postbacks | Add `'unsafe-inline'` to `script-src` in Phase 1; Phase 3+ requires ScriptManager nonce |
| ViewState hidden field | Not a CSP issue — it's in `<input>` not `<script>` | No action needed |
| `ScriptResource.axd` blocked | WebForms serves scripts via this handler | Add `'self'` to `script-src` (already included) |
| Update Panel broken | Uses `__doPostBack` inline | Same as above — nonce ScriptManager |
| Telerik/DevExpress controls | Often inject inline scripts | Contact vendor for nonce support docs |
