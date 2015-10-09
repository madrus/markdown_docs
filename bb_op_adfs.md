# BeeldBank op ADFS aansluiten
## ADFS
#### Functionele omschrijving
Bij het opstarten van de FH portal website, moet de gebruiker zich inloggen om naar de dienstpagina
te kunnen gaan. ADFS zorgt er voor dat hij zich maar één keer hoeft te authenticeren (SSO)
en daarmee wordt zijn autorisatie bepaald en vast gehouden voor alle overige pagina's
die de gebruiker nodig heeft.

#### Technische omschrijving
TODO...

#### PORTAL
##### Portal rollen
We onderscheiden twee Portal rollen:
- Klant: externe logins via Internet
- Medewerker: interne logins op Intranet

##### Portal cookies
Bij het aanloggen op de portal wordt een Portal cookie aangemaakt danwel bijgehouden.
Daarin staat de gewenste taal voor de portal website vastgelegd.

Methode `Default.aspx.cs -> SetUser()` leest het application cookie:
```c#
Authoriser.Current = ADFSWebGebruiker.FindByUserid(userID);
```

##### Logins
- SYS: [`pub_sys2.floraholland.com`](http://pub_sys2.floraholland.com) met `sy018503/Fl0r@Holland!` (testklant 99999)
- ACC: [`pub_acc2.floraholland.com`](http://pub_acc2.floraholland.com) met `ac018503/ecbeheer3` (testklant 99999)

Script om wachtwoord uit te lezen:
```sql
   use el_dbs

   select * from asl_ovk
   where asl_ovk_usr_idf = 'fh018503'
```

##### Website
- fysieke server: `BA03T`
- website oud: `MyFloraHolland-sys-intern`
- domein oud: `ECPRD`
- website nieuw: `MyFloraHolland-sys`
- domein nieuw: `EC`

Het is de bedoeling dat de oude domein en website uitgefaseerd worden en alle websites
gemigreerd zijn naar de nieuwe domein en website

#### BEELDBANK Applicatie
- De applicatie maakt gebruik van **web controls**
- `Default.Master` webpagina is het **standaard template** voor alle applicatie webpagina's.
  Dit template bestaat uit een aantal ContentPlaceHolder elementen die later worden dynamisch
  vervangen door de gewenste content.

##### States
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

##### Cookies
Beeldbank applicatie maakt gebruik van de Application Cookie. Configuratie ervan is web.config te vinden
als de volgende setting:
```c#
<FhAppSettings>
	<All>
		<!-- Cookie met user en language -->
		<add key="appCookieName" value="fh.userlang" />
	</All>
	...
</FhAppSettings>
```

##### Navigatie
De class die verantwoordelijk is voor navigatie tussen verchillende pagina's is **FhNavigation**.
Voorbeeld: `FhNavigator.NavigateTo(FhSession.Startpage);`

#### Gewenst gedrag
- bij het opstarten van Beeldbank in ACC de eeste pagina is [OverzichtPartijBeelden.aspx](https://diensten2-acc.floraholland.com/Beeldbank/Tasks/Am/OverzichtPartijBeelden.aspx)
- bij het opstarten van Beeldbank in LOC de eeste pagina is [Main.aspx](http://localhost:50587/Tasks/Am/Main.aspx)

##### Opstart volgorde in de solution in LOC

Bij het runnen van de solution wordt als eerst deze link in de browser geladen: http://localhost:50587/

###### `Global.asax`
1. `Application_Start()`
2. `IdentityCongig.Initialize()`
  1. `FederatedAuthentication.FederationConfigurationCreated` wordt gekoppeld om de data uit de web.config te lezen
  2. Een reeks handlers wordt geinitialiseerd
  3. Informatie over de verbinding naar de am_dbs database in DEV wordt vastgelegd

###### `Claims.asax`
1. Sessie naar de ADFS authority wordt gestart via http://localhost:8000/STS/Issue
2. Een Literal wordt toegevoegd aan Controls.

###### `Global.asax`
1. `Session_Start()`
  1. In de HttpApplicationState wordt gekeken of er tot dusver exceptions aanwezig zijn. Zo ja, einde oefening.
  2. Zo niet, de sessie wordt opgeschoond d.m.v. `Session.Clear()`
  3. De startpagina wordt bepaald als

     `FhSession.Startpage = AmPageLocation.Default;`

     En dat is `Default.aspx`.

###### `Default.asax`
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

###### `VbaWelcome.asax`
1. `Page_Load -> Authorizer.LoggedOn()`
2. LoggedOn = false
   LoggedOn is true indien gebruiker object gezet is EN `ger_informatierelatie` is gezet.
   Bij vba11397 `ger_informatierelatie = null` en dus false.
3. Bij false wordt `VbaAanmelden` webtask geroepen. Maar we komen in `ExceptionLabel.asmx` terecht.

###### `ExceptionLabel.asax`
1. `Page_Unload -> SetExceptionLabel(null);`

   NB. Page_Unload lijkt elke keer geroepen te worden bij het wisselen van de webpagina.

###### `VbaAanmelden.asax`
1. `Execute()`
2. `var username = ADFSWebGebruiker.GetWebUserName(this.Page);`
3. `TaskStates.New`
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

###### `VbaSelecteerInformatierelatie`
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

###### `Global.asax`
1. `Application_BeginRequest()`
  1. Er wordt ge-faked dat we aangelogd zijn (LoggedOn = true)
  2. De controle gaat door naar 'VbaAanmelden'

###### `VbaAanmelden.asax`
1. `Execute()`
2. LastStatus = New
3. Zie de logica hierboven
4. Door naar `VbaSelecteerInformatierelatie`

###### `VbaSelecteerInformatierelatie`
1. `Execute()`
2. **???**
3. Nogmaals door naar `VbaAanmelden.asax`

###### `VbaAanmelden.asax`
1. `Execute()`
2. LastStatus = OK werd ergens gezet (was New) **TODO: where?**
3. Nu is LastTask = VbaSelecteerInformatierelatie
4. Dan mogen we verder want AD autorisatie is gelukt.

###### `VbaWelkom.asax`
1. `Page_Load()`
2. Deze keer LoggedOn = true
3. `Navigator.Run(AmTasks.Main);`

***
#### Voorbeeld applicaties
Een aantal zaken kunnen bij `XEL` of `VBI` solution bekeken worden hoe het moet. Daar is ADFS al geimplementeerd en werkt.

#### Referentiele klantgegevens
Sommige gegevens zoals de klantnaam worden via een join op `ld_ccn_klt_act` en `ld_ccn_klt` gelezen.

#### ADFS tools
- AD FS op Windows Server 2012

***
## Google Tag Manager (GTM)
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

***
## Gelopen stappen
1. [ASP.NET 2.0 AJAX Extensions 1.0](https://www.microsoft.com/en-us/download/confirmation.aspx?id=883) geinstalleerd
2. `AjaxControlToolkit` uit Visual SourceSafe opgehaald (`$/Ontwikkeling/AZ/AjaxControlToolkit`)
3. GTM script aan `Default.Master` toegevoegd

***
## Testplan
1. Alle links moeten werken
2. Testen met een klantnummer derde (role = 5)
3. Deep links (direct opvoeren in de browser) moeten niet mogelijk zijn en de klant moet
   terug naar de portal login pagina gestuurd worden.
4. Op alle pagina's moet het GTM script aanwezig zijn

***
## Code Snippets
### WebTasks (properties)
```c#
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
```c#
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
***
## Bekende fouten en fixes ervoor
#### VbaScripts.js
##### Error: Uncaught SyntaxError: Unexpected token .
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
##### Fix
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

#### Main.aspx
##### Error: Uncaught ReferenceError: AddTimeoutHandling is not defined
Main.aspx erft van Default.Master.aspx. In Default.Master.cs staat het volgende
```c#
ScriptManager.RegisterClientScriptInclude(this.Page, this.GetType(), "VbaScripts", Global.UrlSitePrefix + "Vba/Scripts/VbaScripts.js");
ScriptManager.RegisterStartupScript(this.Page, this.GetType(), "HandleTimeout", "AddTimeoutHandling();", true);
```
`AddTimeoutHandling` wordt in `VbaScripts` gedefineerd en vervolgens in `HandleTimeout` geroepen.
Waarom komt dat niet naar de afgeleide pagina Main.aspx?

Overigens is dit het geval op alle pagina's, ook in ACC. Deze fouten zijn dus oer oud. :)
##### Fix
Nog **TODO**

##### Error: Main.aspx wordt geladen i.p.v. OverzichtPartijBeelden.aspx
