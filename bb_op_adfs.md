BeeldBank op ADFS aansluiten
============================
---
Functionele omschrijving
------------------------

Bij het opstarten van de FH portal website, moet de gebruiker zich inloggen om naar 
de dienstenpagina te kunnen gaan. ADFS zorgt er voor dat hij zich maar één keer hoeft 
te authenticeren (SSO) en daarmee wordt zijn autorisatie bepaald en vastgesteld voor 
alle overige pagina's die de gebruiker nodig heeft.

Dit zijn de drie mogelijke workflows (scenario's) van het autorisatieproces

### Externe klant
1. Klant logt op de [FH portal](https://www.floraholland.com/nl/) aan
2. Hij gaat naar `Mijn FloraHolland` pagina en kiest voor bijv. `Start webdiensten | Beeldbank voor kwekers`
3. Als hij een koper of een aanvoerder is, wordt de Beeldbank applicatie gestart 
   en krijgt hij de `Overzicht Foto's` pagina te zien

### FH medewerker
1. FH medewerker logt in op bijv. [`pub_sys2.floraholland.com`](http://pub-sys2.floraholland.com/nl/)
   onder zijn eigen FH login
2. Vervolgens kiest hij de klant waarvoor hij `Beeldbank` dienst wil bekijken.
3. Als deze klant meer dan één relaties heeft, kiest de medewerker de gewenste relatie 
   en krijgt hij de `Overzicht Foto's` pagina te zien.

### AO&IA ontwikkelaar
Als ontwikkelaar draai je de solution met `F5` of `Ctrl+F5` en dan begin je als het ware 
net als FH medewerker maar dan gelijk bij punt 2.

---
## Autorisatie procedure
---

Er zijn drie mogelijke workflows (scenario's) van het autorisatieproces

### Externe klant
1. Klant logt op de [FH portal](https://www.floraholland.com/nl/) aan
2. Hij gaat naar `Mijn FloraHolland` pagina en kiest voor bijv. `Start webdiensten | Beeldbank voor kwekers`
3. `AM applicatie` wordt gestart en `Default.aspx.cs` in zijn `SetUser()` methode ontvangt
   `ApplicationCookie` uit `HttpContext.Current` via
> `var ac = CookieHelper.GetApplicationCookie();`

  `ApplicationCookie` ziet er als volgt uit:  
```csharp
[Serializable]
public class ApplicationCookie
{
    public string IsoLang { get; set; }
    public string ApplicationUser { get; set; }
    public string ApplicationUrl { get; set; }
}
```
4. Deze cookie heeft in `Web.config` als naam `"fh.userlang"`.
5. De waarde van `ApplicationUser` wordt `userID`. Voor de externe klant heeft deze waarde
   een vast formaat als `'fh018503'` of `'vba07248'`.
6. Vervolgens wordt op basis van de `userID` de `ADFSGebruiker` bepaald via 
> `ADFSWebGebruiker.FindByUserid(ac.ApplicationUser);`

   In de `ADFSWebGebruiker` zit nu het `klantnummer` en de `klantrolcode` die nodig zijn
   om te bepalen of deze gebruiker de `Beeldbank` applicatie mag gebruiken.

### FH medewerker
1. De authorisatieproces voor een FH medewerker is identiek aan dat van een externe klant.
   Het verschil is dat de `userID` wordt in dit geval het FH login, bijv. `ar` 

### AO&IA ontwikkelaar
1. Als je de solution met `F5` of `Ctrl+F5` start, wordt je via een default cookie geautoriseerd.

### Meer informatie
Hier is meer informatie te vinden over de [ASP.NET Page Life Cycle](https://msdn.microsoft.com/en-us/library/vstudio/ms178472%28v=vs.100%29.aspx)

---
## FloraHolland Portal
---

### Portal rollen
We onderscheiden twee Portal rollen:
- `Klant`: externe logins via Internet
- `Medewerker`: interne logins op Intranet

### Portal cookies
Bij het aanloggen op de portal wordt een Portal cookie aangemaakt danwel bijgehouden.
Daarin staat de gewenste taal voor de portal website vastgelegd.

Methode `Default.aspx.cs -> SetUser()` leest het application cookie:
```csharp
Authoriser.Current = ADFSWebGebruiker.FindByUserid(userID);
```

### Logins
- SYS: [`pub_sys2.floraholland.com`](http://pub-sys2.floraholland.com/nl/) met `sy018503/Fl0r@Holland!` (testklant 99999)
- ACC: [`pub_acc2.floraholland.com`](http://pub_acc2.floraholland.com) met `ac018503/ecbeheer3` (testklant 99999)

Script om wachtwoord uit te lezen:
```sql
use el_dbs

select * from asl_ovk
where asl_ovk_usr_idf = 'fh018503'
```

### Website
* server: `BA03T`
* oude situatie
  * website: `MyFloraHolland-sys-intern`
  * domein: `ECPRD`
* nieuwe situatie
  * website: `MyFloraHolland-sys`
  * domein: `EC`

Het is de bedoeling dat de oude domein en website uitgefaseerd worden en alle websites
gemigreerd zijn naar de nieuwe domein en website

---
## BeeldBank Applicatie
---

- De applicatie maakt gebruik van **web controls**
- `Default.Master` webpagina is het **standaard template** voor alle applicatie webpagina's.
  Dit template bestaat uit een aantal ContentPlaceHolder elementen die later worden dynamisch
  vervangen door de gewenste content.

### States
Informatie wordt tussentijds opgeslagen in de state objects van deze types:
- `HttpContext` om de cookies te lezen
- `StateManager` wordt gebruikt bij de **ADFS autorisatie** van de gebruiker maar ook om zijn
  gebruikersinformatie, bijv. zijn taalvoorkeur of dat hij meerdere klantrelaties heeft,
  vast te houden
- `Session` wordt gebruikt om de data van de specifieke schermen vast te houden
- **NB:** Het lijkt erop dat er geen gebruik wordt gemaakt van Application State, so de data
  is enkel beschikbaar binnen één enkel browser scherm.
- `DaVinci.Process.TaskStates` worden gebruikt om de acties vd gebruiker bij te houden,
  bijv. toevoegen of wijzigen van een foto (klopt dat?)

### Cookies
Beeldbank applicatie maakt gebruik van de Application Cookie. Configuratie ervan is 
in `web.config` te vinden als de volgende setting:
```csharp
<FhAppSettings>
    <All>
        <!-- Cookie met user en language -->
        <add key="appCookieName" value="fh.userlang" />
    </All>
    ...
</FhAppSettings>
```

### Navigatie
De class die verantwoordelijk is voor navigatie tussen verchillende pagina's is **FhNavigation**.
Voorbeeld: `FhNavigator.NavigateTo(FhSession.Startpage);`

---
## Gewenst gedrag
---

- bij het opstarten van Beeldbank in ACC de eeste pagina is [OverzichtPartijBeelden.aspx](https://diensten2-acc.floraholland.com/Beeldbank/Tasks/Am/OverzichtPartijBeelden.aspx)
- bij het opstarten van Beeldbank in LOC de eeste pagina is [Main.aspx](http://localhost:50587/Tasks/Am/Main.aspx)

### Opstart volgorde in de solution in LOC

Bij het runnen van de solution wordt als eerst deze link in de browser geladen: http://localhost:50587/

#### `Global.asax`
1. `Application_Start()`
2. `IdentityConfig.Initialize()`
  1. `FederatedAuthentication.FederationConfigurationCreated` event handler wordt gekoppeld om de data uit de web.config te lezen
  2. Per omgeving wordt een reeks properties gezet op `FederationConfiguration.WsFederationConfiguration`
     op basis van web.config:
     * Issuer
     * Realm
     * Reply
     * RequireHttps
  3. Informatie over de verbinding naar de am_dbs database in DEV wordt vastgelegd

#### `Claims.asax`
1. Sessie naar de ADFS authority wordt gestart via http://localhost:8000/STS/Issue
2. Een Literal wordt toegevoegd aan Controls.

#### `Global.asax`
1. `Session_Start()`
  1. In de HttpApplicationState wordt gekeken of er tot dusver exceptions aanwezig zijn. Zo ja, einde oefening.
  2. Zo niet, de sessie wordt opgeschoond d.m.v. `Session.Clear()`
  3. De startpagina wordt bepaald als

    `FhSession.Startpage = AmPageLocation.Default;`

    En dat is `Default.aspx`.

#### `Default.asax`
1. `Page_Load -> SetUser()`
2. In de uit de HttpContext gelezen cookie zie ik twee keys: **`csrftoken`** en **`FedAuth`**
   Echter geen **`fh.userlang`** die in de `web.config` staat gedefineerd.
   De applicatioCookie blijft daardoor null.
3. Daardoor kunnen we de userID niet bepalen en wordt de ADFS autorisatie geprobeerd op basis
   van de claims identity in de HttpContext:

   `Authoriser.Current = ADFSWebGebruiker.FindByUserid(HttpContext.Current.User.Identity.Name);`

   Daar staat `name=vba11397` vermeld. Deze claim was uitgegeven door **SelfSTS**:

   `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name: vba11397`
   `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/rol: klant`

4. De Navigator wordt opgeschoond d.m.v. `Navigator.Clear()`
5. De Navigator start VbaWelkom task: `Navigator.Run(WebTasks.VbaWelkom);`

### `ADFSWebGebruiker`
#### `FindByUserid`
1. Checkt of `web.config` een dummy authorisatie bevat (`ADDummyInfo` setting).
2. Als deze bestaat, wordt hij overgenomen. Zo niet, dan op **line 301** staat
   een **TODO -- naar de database** vermeld doch **niet geimplementeerd**.
3. Dan wordt een gebruiker aangemaakt en teruggeven met de volgende waarden:
```csharp
public const int DEFAULT_KLANTNUMMER = 0;
public const int DEFAULT_KLANTROLKODE = 3;
public const int DEFAULT_KLANTRELATIENUMMER = 17007;
var adfs = new ADFSWebGebruiker(userid, "", "", "", DEFAULT_KLANTNUMMER,
                  DEFAULT_KLANTROLKODE, DEFAULT_KLANTRELATIENUMMER, "", null);
```

#### `IsPersoneel`
**Altijd FALSE**

#### `VbaWelcome.asax`
1. `Page_Load -> Authorizer.LoggedOn()`
2. LoggedOn = false
   LoggedOn is true indien gebruiker object gezet is EN `ger_inforelatie` is gezet.
   Bij vba11397 `ger_inforelatie = null` en dus false.
3. Bij false wordt `VbaAanmelden` webtask geroepen. Maar we komen in `ExceptionLabel.asmx` terecht.

#### `VbaAanmelden.asax`
1. `Execute()`
2. Er wordt `bool VbaWebADFSAuthoriser.Login(string login, string password)` geroepen
   die **altijd TRUE** teruggeeft.
3. `var username = ADFSWebGebruiker.GetWebUserName(this.Page);`
4. `TaskStates.New`
   1. Gebruiker is leeg => niet geauthoriseerd
   2. Geen wachtwoord => niet geauthoriseerd
   3. Dus geauthoriseerd
      1. Is informatierelatie niet van belang? Klaar.
      2. Dus wel van belang. Is gebruiker een FH medewerker?
         1. ja wel -> kijk in de config of impersonatie gewenst is (**TODO: uitzoeken**)
            1. nee -> geen impersonatiescherm tonen en klaar.
            2. ja -> toon dan impersonatiescherm en laat de klantrelatie kiezen.
               M.a.w. roep 'VbaSelecteerKlantrelatie' zonder parameters en klaar.
         2. nee hoor -> roep 'VbaSelecteerInformatierelatie' aan met klantrolkode en klantrelatienummer

#### `VbaSelecteerInformatierelatie`
1. `Execute()`
2. Bij een nieuwe webtask wordt geprobeerd om de `Informatierelatie` setting uit web.config
   op te halen met als waarde `on` of `off`.
3. Als dat wenselijk is, worden relaties geraadpleegd:

   `relaties = Relatie.Zoek(KlantRelatierol, KlantRelatienummer);`

   Daarvan afhankelijk wordt status gezet:

   `StateManager.User[AmStates.MeerdereRelaties] = relaties.Length > 1;`

   Dan wordt gekeken naar het aantal.
   1. geen -> logout en ga terug naar login
   2. één -> deze wordt gekozen en teruggegeven
   3. meer dan één -> vul en toon een combo met relaties

#### `Global.asax`
1. `Application_BeginRequest()`
  1. Er wordt ge-faked dat we aangelogd zijn (LoggedOn = true)
  2. De controle gaat door naar 'VbaAanmelden'

#### `VbaAanmelden.asax`
1. `Execute()`
2. LastStatus = New
3. Zie de logica hierboven
4. Door naar `VbaSelecteerInformatierelatie`

#### `VbaSelecteerInformatierelatie`
1. `Execute()`
2. **???**
3. Nogmaals door naar `VbaAanmelden.asax`

#### `VbaAanmelden.asax`
1. `Execute()`
2. LastStatus = OK werd ergens gezet (was New) **TODO: where?**
3. Nu is LastTask = VbaSelecteerInformatierelatie
4. Dan mogen we verder want AD autorisatie is gelukt.

#### `VbaWelkom.asax`
1. `Page_Load()`
2. Deze keer LoggedOn = true
3. `Navigator.Run(AmTasks.Main);`

---
## Voorbeeld applicaties
---

Een aantal zaken kunnen bij `XEL` of `VBI` solution bekeken worden hoe het moet. Daar is ADFS al geimplementeerd en werkt.

---
## Referentiele klantgegevens
--- 

Sommige gegevens zoals de klantnaam worden via een join op `ld_ccn_klt_act` en `ld_ccn_klt` gelezen.

---
## ADFS tools
---

- AD FS op Windows Server 2012

---
## Google Tag Manager (GTM)
---

### FloraHolland GTM Snippet
De onderstaande snippet moet op elke webpagina van de BeeldBank applicatie geplaatst worden
direct naar de open `<body>` tag.
```js
<!-- Google Tag Manager -->
<noscript>
    <iframe src="//www.googletagmanager.com/ns.html?id=GTM-T2KW75" height="0" width="0" style="display: none; visibility: hidden"></iframe>
</noscript>
<script type="text/javascript">
    (function (w, d, s, l, i) {
        w[l] = w[l] || []; w[l].push({ 'gtm.start': new Date().getTime(), event: 'gtm.js' });
        var f = d.getElementsByTagName(s)[0], j = d.createElement(s), dl = l != 'dataLayer' ? '&l=' + l : '';
        j.async = true; j.src = '//www.googletagmanager.com/gtm.js?id=' + i + dl; f.parentNode.insertBefore(j, f);
    })(window, document, 'script', 'dataLayer', 'GTM-T2KW75');
</script>
<!-- End Google Tag Manager -->
```

---
## Gelopen stappen
---

1. [ASP.NET 2.0 AJAX Extensions 1.0](https://www.microsoft.com/en-us/download/confirmation.aspx?id=883) geinstalleerd
2. `AjaxControlToolkit` uit Visual SourceSafe opgehaald (`$/Ontwikkeling/AZ/AjaxControlToolkit`)
3. GTM script aan `Default.Master` toegevoegd
4. ADFSWebGebruiker class gefixt (properties)
5. WebGebruiker overal door ADFSWebGebruiker vervangen
6. View CldKlantAccount in am_dbs toegevoegd (kopieconform VbiMain)
7. Class CldKlantAccount aan Vba.Am.Business toegevoegd met getters
8. Class TableCldKlantAccount aan Vba.Am.Data toegevoegd met getters
9. View CldKlant in am_dbs toegevoegd (kopieconform VbiMain)
10. Class CldKlant aan Vba.Am.Business toegevoegd met getters
11. Class TableCldKlant aan Vba.Am.Data toegevoegd met getters
12. Testproject Vba.Am.Web.Test toegevoegd voor unit tests

---
## Testplan
---

1. Alle links moeten werken
2. Testen met een klantnummer derde (role = 5)
3. Deep links (direct opvoeren in de browser) moeten niet mogelijk zijn en de klant moet
   terug naar de portal login pagina gestuurd worden.
4. Op alle pagina's moet het GTM script aanwezig zijn

---
## Code Snippets
---

### WebTasks (properties)
```csharp
namespace Vba.Az.DaVinci.Web.Process
{
    public class WebTasks : Tasks
    {
        public static WebTasks VbaAanmelden;
        public static WebTasks VbaInfo;
        public static WebTasks VbaMessageNotAutorised;
        public static WebTasks VbaOnderhoudenhtmltekst;
        public static WebTasks VbaSelecteerInformatierelatie;
        public static WebTasks VbaSelecteerKlantrelatie;
        public static WebTasks VbaVragen;
        public static WebTasks VbaWelkom;
        public static WebTasks VbaZoekenhtmltekst;
    }
}
```
### AmTasks (properties)
```csharp
namespace Vba.Am.Web
{
    public class AmTasks : DaVinci.Process.Tasks
    {
        // Klant
        public static AmTasks Main = new AmTasks("Main", "Tasks/Am/Main");
        public static AmTasks OverzichtPartijBeelden = new AmTasks("OverzichtPartijBeelden", "Tasks/Am/OverzichtPartijBeelden");
        public static AmTasks DetailsPartijBeeld = new AmTasks("DetailsPartijBeeld", "Tasks/Am/DetailsPartijBeeld");
        public static AmTasks ToevoegenProductcode = new AmTasks("ToevoegenProductcode", "Tasks/Am/ToevoegenProductcode");
        public static AmTasks SelecteerProductKenmerk = new AmTasks("SelecteerProductKenmerk", "Tasks/Am/SelecteerProductKenmerk");
        public static AmTasks SelecteerProductKenmerkWaarde = new AmTasks("SelecteerProductKenmerkWaarde", "Tasks/Am/SelecteerProductKenmerkWaarde");
        public static AmTasks ToevoegenFoto = new AmTasks("ToevoegenFoto", "Tasks/Am/ToevoegenFoto");
        public static AmTasks ToevoegenFotoMelding = new AmTasks("ToevoegenFotoMelding", "Tasks/Am/ToevoegenFotoMelding");
        public static AmTasks ToevoegenKenmerk = new AmTasks("ToevoegenKenmerk", "Tasks/Am/ToevoegenKenmerk");
        public static AmTasks FotoResizedVoorbeeld = new AmTasks("FotoResizedVoorbeeld", "Tasks/Am/FotoResizedVoorbeeld");
        public static AmTasks Bloemenklok = new AmTasks("Bloemenklok", "Tasks/Am/Bloemenklok");
        public static AmTasks Plantenklok = new AmTasks("Plantenklok", "Tasks/Am/Plantenklok");

        // Personeel
        public static AmTasks OverzichtAanvoerder = new AmTasks("OverzichtAanvoerder", "Tasks/Personeel/OverzichtAanvoerder");
        public static AmTasks OverzichtPlanten = new AmTasks("OverzichtPlanten", "Tasks/Personeel/OverzichtPlanten");
        public static AmTasks OverzichtBloemen = new AmTasks("OverzichtBloemen", "Tasks/Personeel/OverzichtBloemen");
        public static AmTasks OverzichtMutaties = new AmTasks("OverzichtMutaties", "Tasks/Personeel/OverzichtMutaties");
        public static AmTasks OverzichtBeelden = new AmTasks("OverzichtBeelden", "Tasks/Personeel/OverzichtBeelden");
        public static AmTasks BeeldcontroleAanvoerder = new AmTasks("BeeldcontroleAanvoerder", "Tasks/Personeel/BeeldcontroleAanvoerder");
        public static AmTasks BeeldcontroleAanvoerderWijzigen = new AmTasks("BeeldcontroleAanvoerderWijzigen", "Tasks/Personeel/BeeldcontroleAanvoerderWijzigen");
        public static AmTasks OverzichtBeeldenPerProductcode = new AmTasks("OverzichtBeeldenPerProductcode", "Tasks/Personeel/OverzichtBeeldenPerProductcode");

        // Support
        public static AmTasks VraagAntwoord = new AmTasks("VraagAntwoord", "Tasks/Am/VraagAntwoord");
        public static AmTasks Servicedesk = new AmTasks("Servicedesk", "Tasks/Am/Servicedesk");

        //Diversen/algemeen:
        public static AmTasks ToExcel = new AmTasks("ToExcel", "Tasks/Public/ToExcel");
    }
}
```

---
## Bekende fouten en fixes ervoor
---

### VbaScripts.js
#### Error: Uncaught SyntaxError: Unexpected token
Het gaat om het punt tussen window en de functienaam.
```js
// verberg bepaalde elementen vooraf aan het printen
function window.onbeforeprint()
{
    updatePrintLayout('none');
}

// maak elementen weer zichtbaar na het printen
function window.onafterprint()
{
    updatePrintLayout('');
}
```
#### Fix
Het syntax is niet correct omdat onbeforeprint en onafterprint zijn windows properties.
Hieronder heb ik mijn fix geplaatst.
```js
// verberg bepaalde elementen vooraf aan het printen
window.onbeforeprint = function()
{
    updatePrintLayout('none');
}

// maak elementen weer zichtbaar na het printen
window.onafterprint = function()
{
    updatePrintLayout('');
}
```

### Main.aspx
#### Error: Uncaught ReferenceError: AddTimeoutHandling is not defined
Main.aspx erft van Default.Master.aspx. In Default.Master.cs staat het volgende
```csharp
ScriptManager.RegisterClientScriptInclude(this.Page, this.GetType(), "VbaScripts", Global.UrlSitePrefix + "Vba/Scripts/VbaScripts.js");
ScriptManager.RegisterStartupScript(this.Page, this.GetType(), "HandleTimeout", "AddTimeoutHandling();", true);
```
`AddTimeoutHandling` wordt in `VbaScripts` gedefineerd en vervolgens in `HandleTimeout` geroepen.
Waarom komt dat niet naar de afgeleide pagina Main.aspx?

Overigens is dit het geval op alle pagina's, ook in ACC. Deze fouten zijn dus oer oud. :)
#### Fix
**TODO**

#### Error: Main.aspx wordt geladen i.p.v. OverzichtPartijBeelden.aspx

---
##Vragen
---

1. Wat is het verschil tussen KlantRelatie en InformatieRelatie?
   * InformatieRelatie is een klant waarvoor de KlantRelatie mag beelden inzien.
   * KlantRelatie is een klant die bijv. door de FH medewerker geselecteerd worden
     om testdoeleinden of om te helpen
2. nog geen volgende vraag

---
## Analyse van de solution
---

### ADFS en ADFSWebGebruiker komen voor in de volgende bestanden
- Vba.Am.Web/Autorisation/ADFSWebGebruiker.cs
- Vba.Am.Web/Autorisation/VbaWebADFSAuthoriser.cs
- Vba.Am.Web/Default.aspx.cs
- Vba.Am.Web/Global.asax.cs
- Vba.Am.Web/IdentityConfig.cs
- Vba.Am.Web/Vba/TasksAlgemeen/VbaAanmelden.aspx.cs
- Vba.Am.Web/Vba/TasksAlgemeen/VbaSelecteerInformatierelatie.aspx.cs
- Vba.Am.Web/Vba/UserControls/MenuAlgemeen.ascx.cs
- Vba.Am.Web/Vba/UserControls/MenuOverige.ascx.cs
- Vba.Am.Web/Vba/UserControls/MenuSpecifiek.ascx.cs
- Vba.Am.Web/Vba/UserControls/SelecteerKlantRelatie.ascx.cs
- Vba.Am.Web/Web.config

### WebGebruiker komt voor in de volgende bestanden
- Vba.Am.Web/App_GlobalResources/vba.resources.vbastrings.en.resx
- Vba.Am.Web/App_GlobalResources/vba.resources.vbastrings.nl.resx
- Vba.Am.Web/Autorisation/ADFSWebGebruiker.aspx.cs
- Vba.Am.Web/Autorisation/VbaWebADFSAuthoriser.aspx.cs
- Vba.Am.Web/Tasks/Am/DetailsPartijBeeld.ascx.cs
- Vba.Am.Web/Tasks/Am/OverzichtPartijBeelden.ascx.cs
- Vba.Am.Web/Utilities.cs
- Vba.Am.Web/Vba/TasksAdmin/VbaOnderhoudenhtmltekst.aspx.cs
- Vba.Am.Web/Vba/TasksAlgemeen/VbaSelecteeerInformatieRelatie.aspx.cs
- Vba.Am.Web/Vba/UserControls/Info.ascx.cs
- Vba.Am.Web/Vba/UserControls/Vragen.ascx.cs
-