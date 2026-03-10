**(Portuguese)**

**Introdução ao CAPTCHA no SAP Commerce Cloud**

Antes de implementar o CAPTCHA no frontend, é importante entender o que é o CAPTCHA, como ele funciona e como é configurado no SAP Commerce Cloud.

Você pode encontrar uma explicação detalhada na documentação abaixo: 

https://github.com/viniciusgugelmin/personal-guide/tree/master/src/SAP%20Commerce%20Cloud/Captcha

**Pré-requisitos**

Antes de iniciar a implementação no frontend, certifique-se de que os seguintes pré-requisitos foram atendidos:
- As chaves do Google reCAPTCHA foram criadas
- O CAPTCHA está configurado no SAP Commerce Cloud
- As chaves pública e privada estão disponíveis no ambiente Commerce
- As configurações de IMPEX necessárias foram executadas

Sem essas etapas, o frontend não receberá a configuração do CAPTCHA.

**Fluxo de Alto Nível**

O fluxo do CAPTCHA no Spartacus + SAP Commerce Cloud funciona da seguinte forma:

	Página do Spartacus
	     ↓
	Componente <cx-captcha>
	     ↓
	CustomCaptchaService
	     ↓
	API do Google reCAPTCHA
	     ↓
	Token gerado
	     ↓
	Token enviado no cabeçalho da requisição
	sap-commerce-cloud-captcha-token
	     ↓
	Endpoint OCC
	     ↓
	CaptchaValidationInterceptor
	     ↓
	CaptchaValidationService

Após a validação bem-sucedida, a requisição é processada normalmente pelo SAP Commerce Cloud.

**Primeiros Passos**

Para iniciar a implementação do CAPTCHA no lado do frontend, é necessário apenas garantir que os seguintes arquivos IMPEX tenham sido executados.

Essas configurações garantem que:
- o site exponha as chaves do CAPTCHA para o frontend
- o frontend possa enviar o token do CAPTCHA para os endpoints OCC sem erros de CORS

```
	INSERT_UPDATE BaseStore; uid[unique=true]; captchaCheckEnabled
	; $storeUid ; true

	INSERT_UPDATE CorsConfigurationProperty; context[unique = true]; key[unique = true]; value
	; commercewebservices ; allowedHeaders ; origin content-type accept authorization cache-control if-none-match x-anonymous-consents x-profile-tag-debug x-consent-reference occ-personalization-id occ-personalization-time Sap-Commerce-Cloud-User-Id sap-commerce-cloud-captcha-token
```

**Verificando a Configuração do CAPTCHA**

Após executar os arquivos IMPEX, abra a página inicial e inspecione a requisição do BaseSite.

A resposta deve incluir a seguinte estrutura:

	{
	  "...": "...",
	  "captchaConfig": {
	    "enabled": true,
	    "publicKey": "<PUBLIC_KEY>"
	  }
	}

Esta configuração fica disponível dentro do CaptchaService do Spartacus.

**Iniciando a Implementação no Frontend**

Para implementar o CAPTCHA no frontend, é necessário criar um serviço customizado que estenda o CaptchaService do Spartacus.

Isso é necessário porque o CaptchaService no Spartacus é uma classe abstrata, o que significa que não pode ser instanciado diretamente e deve ser estendido.

Exemplo:

	CustomCaptchaService extends CaptchaService

**Registrando o Serviço de CAPTCHA Customizado**

Antes de implementar o comportamento do CAPTCHA, é necessário registrar o serviço customizado no módulo onde o CAPTCHA será utilizado.

Dentro dos providers do módulo:

```javascript annotate
	provideConfig(<CaptchaApiConfig>{
	  captchaRenderer: CustomCaptchaService,
	})
```
	
Isso instrui o Spartacus a usar a implementação customizada em vez da padrão.

**Adicionando CAPTCHA a um Componente**

Após registrar o serviço, o CAPTCHA pode ser renderizado dentro do template de qualquer componente.

Exemplo:

	<cx-captcha (confirmed)="onCaptchaConfirmed()"></cx-captcha>
	
Este componente acionará o fluxo do CAPTCHA e notificará o componente quando a validação for bem-sucedida.

**Customizando o Comportamento do CAPTCHA**

Para criar uma experiência de CAPTCHA simples, porém amigável ao usuário, alguns métodos do CaptchaService devem ser sobrescritos.

Adicionalmente, uma lógica auxiliar é introduzida para melhorar:
- o feedback do usuário
- o tratamento de expiração de token
- o gerenciamento de estado da UI

**Adicionando Traduções**

Para suportar a internacionalização, as traduções devem ser adicionadas ao módulo de tradução do Spartacus.

Exemplo:

```javascript annotate
	export const <SPARTACUS_TRANSLATION_MODULE> = {
	  captcha: {
	    succeed: 'Sucesso',
	    verified: 'Verificado',
	    robot: 'Não sou um robô',
	  }
	}
```

**Carregando Traduções no Serviço**

Dentro do construtor do CustomCaptchaService, colocamos subscribe nos valores de tradução usando o TranslationService.

Isso permite atualizações dinâmicas de tradução quando o idioma é alterado.

```javascript annotate
	this.translationSubscription = combineLatest([
	  this.translationService.translate('captcha.succeed'),
	  this.translationService.translate('captcha.verified'),
	  this.translationService.translate('captcha.robot')
	]).pipe(
	  map(([succeed, verified, robot]) => {
	    this.succeed = succeed;
	    this.verified = verified;
	    this.robot = robot;
	  })
	).subscribe();
```

Para esta implementação, certifique-se de injetar o TranslationService e declarar as variáveis:
- succeed
- verified
- robot

**Sobrescrevendo o Método initialize**

O método initialize() é responsável por carregar a configuração do CAPTCHA e preparar a UI.

Aqui usamos o método nativo do Spartacus:

```javascript annotate
	fetchCaptchaConfigFromServer()
```

Isso recupera a configuração do SAP Commerce.

```javascript annotate
	override initialize() {
	  this.subscription.add(
	    this.fetchCaptchaConfigFromServer().subscribe((config) => {
	      if (config?.enabled) {
		this.captchaConfig = config;
		this.loadResource();
	      } else {
		this.captchaConfigSubject$.next({ enabled: false });
		this.captchaRequired$.next(false);
	      }
	    })
	  );
	}
```

Esta lógica garante que:
- o CAPTCHA seja renderizado apenas quando habilitado no Commerce
- permaneça desabilitado caso contrário

A variável captchaRequired$ deve ser definida como:

```javascript annotate
	BehaviorSubject<boolean>
```

Isso controla se a validação do CAPTCHA é obrigatória.

**Renderizando a UI do CAPTCHA**

Dentro do mesmo método initialize(), criamos dinamicamente uma UI baseada em checkbox.

Este checkbox simula uma interação simples de "Não sou um robô".

```javascript annotate
	this.container = document.createElement('div');
	this.container.className = 'form-check';

	this.checkbox = document.createElement('input');
	this.checkbox.type = 'checkbox';
	this.checkbox.id = 'custom-recaptcha-checkbox';

	this.label = document.createElement('label');
	this.label.textContent = this.robot;
	this.label.htmlFor = this.checkbox.id;

	this.container.appendChild(this.checkbox);
	this.container.appendChild(this.label);

	this.spinner = document.createElement('icon');
	this.spinner.className = 'fa-solid fa-spinner';

	this.checkbox.addEventListener('change', this.onCheckBoxClicked.bind(this));
```

Novas variáveis introduzidas:
- container
- checkbox
- label
- spinner

Estas são usadas para construir a UI dinâmica.

**Lidando com o Clique no CAPTCHA**

O método onCheckBoxClicked() aciona a validação do CAPTCHA.

Este método:
- carrega o Google reCAPTCHA
- desabilita o checkbox
- exibe um spinner
- inicia a geração do token

```javascript annotate
	onCheckBoxClicked(): void {
	  this.grecaptcha();
	  this.label.textContent = '';
	  this.container.appendChild(this.spinner);
	  this.checkbox.disabled = true;
	  this.checkbox.checked = true;
	  this.label.className = 'disabled';
	  this.retVal.next(this.succeed);
	  this.retVal.complete();
	}
```

**Sobrescrevendo o renderCaptcha**

O método renderCaptcha() é responsável por anexar a UI do CAPTCHA ao DOM.

```javascript annotate
	override renderCaptcha(renderParams: RenderParams): Observable<string> {
	  if (renderParams.element instanceof HTMLElement) {
	    this.checkbox.disabled = false;
	    this.checkbox.checked = false;
	    this.label.textContent = this.robot;
	    this.label.className = '';
	    this.retVal = new Subject<string>();

	    renderParams.element.appendChild(this.container);
	  }

	  return this.retVal.asObservable();
	}
```

Isso garante que toda vez que o CAPTCHA for renderizado:
- a UI seja resetada
- o checkbox seja habilitado
- uma nova resposta observável seja criada

**Sobrescritas Adicionais Necessárias**

Alguns métodos adicionais devem ser sobrescritos para suportar o comportamento customizado.

Retorna o token gerado pelo CAPTCHA.

```javascript annotate
	override getToken(): string {
	  return this.token;
	}
```

Garante que as inscrições (subscriptions) sejam limpas para evitar vazamentos de memória.

```javascript annotate
	override ngOnDestroy(): void {
	  if (this.translationSubscription) {
	    this.translationSubscription.unsubscribe();
	  }
	}
```

**Lidando com a Expiração do Token do CAPTCHA**

Os tokens do Google reCAPTCHA expiram após um curto período de tempo.

Se o usuário validar o CAPTCHA mas demorar a enviar o formulário, o token pode se tornar inválido.

Para melhorar a experiência do usuário, um manipulador customizado foi implementado:
- exibe uma mensagem de erro
- notifica o usuário
- recarrega a página para reiniciar o processo

Esta lógica é implementada em:

```javascript annotate
	verificationExpired()
```

**Serviço de CAPTCHA Customizado (Código Completo)**

```javascript annotate
	@Injectable({
	  providedIn: 'root'
	})
	export class CustomCaptchaService extends CaptchaService implements OnDestroy {
	  protected translationService: TranslationService = inject(TranslationService);
	  protected globalMessageService = inject(GlobalMessageService);

	  protected retVal: Subject<string> = new Subject<string>();
	  protected captchaEnabledSubject: BehaviorSubject<boolean> = new BehaviorSubject<boolean>(false);
	  public captchaEnabled$ = this.captchaEnabledSubject.asObservable();
	  public captchaRequired$ = new BehaviorSubject<boolean>(true);
	  protected container!: HTMLDivElement;
	  protected checkbox!: HTMLInputElement;
	  protected label!: HTMLLabelElement;
	  protected spinner!: HTMLElement;

	  private readonly TIMEOUT_MESSAGE = 5000;
	  private readonly TOKEN_EXPIRED = 120000;

	  private succeed = '';
	  private verified = '';
	  private robot = '';
	  private translationSubscription: any;

	  constructor() {
	    super(
	      inject(SiteAdapter),
	      inject(CaptchaApiConfig),
	      inject(LanguageService),
	      inject(ScriptLoader),
	      inject(BaseSiteService)
	    );

	    this.translationSubscription = combineLatest([
	      this.translationService.translate('captcha.succeed'),
	      this.translationService.translate('captcha.verified'),
	      this.translationService.translate('captcha.robot')
	    ]).pipe(
	      map(([succeed, verified, robot]) => {
		this.succeed = succeed;
		this.verified = verified;
		this.robot = robot;
	      })
	    ).subscribe();
	  }

	  override initialize() {
	    this.subscription.add(
	      this.fetchCaptchaConfigFromServer().subscribe((config) => {
		if (config?.enabled) {
		  this.captchaConfig = config;
		  this.loadResource();
		} else {
		  this.captchaConfigSubject$.next({ enabled: false });
		  this.captchaRequired$.next(false);
		}
	      })
	    );

	    this.container = document.createElement('div');
	    this.container.className = 'form-check';

	    this.checkbox = document.createElement('input');
	    this.checkbox.type = 'checkbox';
	    this.checkbox.id = 'custom-recaptcha-checkbox';

	    this.label = document.createElement('label');
	    this.label.textContent = this.robot;
	    this.label.htmlFor = this.checkbox.id;
	    this.container.appendChild(this.checkbox);
	    this.container.appendChild(this.label);

	    this.spinner = document.createElement('icon');
	    this.spinner.className = 'fa-solid fa-spinner';

	    this.checkbox.addEventListener('change', this.onCheckBoxClicked.bind(this));
	  }

	  onCheckBoxClicked(): void {
	    this.grecaptcha();
	    this.label.textContent = '';
	    this.container.appendChild(this.spinner);
	    this.checkbox.disabled = true;
	    this.checkbox.checked = true;
	    this.label.className = 'disabled';
	    this.retVal.next(this.succeed);
	    this.retVal.complete();
	  }

	  private grecaptcha() {
	    const publicKey = this.captchaConfig.publicKey;

	    this.scriptLoader.embedScript({
	      src: 'https://www.google.com/recaptcha/api.js?render=' + publicKey
	    });

	    const checkGrecaptcha = () => {
	      // @ts-ignore
	      if (typeof grecaptcha !== 'undefined' && grecaptcha.ready) {
		// @ts-ignore
		grecaptcha.ready(() => {
		  // @ts-ignore
		  grecaptcha.execute(publicKey, { action: 'submit' }).then((token: string) => {
		    this.token = token;
		    this.onTokenSuccess();
		    setTimeout(() => {
		      this.verificationExpired();
		    }, this.TOKEN_EXPIRED);
		  });
		});
	      } else {
		setTimeout(checkGrecaptcha, 100);
	      }
	    };

	    checkGrecaptcha();
	  }

	  override renderCaptcha(renderParams: RenderParams): Observable<string> {
	    if (renderParams.element instanceof HTMLElement) {
	      this.checkbox.disabled = false;
	      this.checkbox.checked = false;
	      this.label.textContent = this.robot;
	      this.label.className = '';
	      this.retVal = new Subject<string>();

	      renderParams.element.appendChild(this.container);
	    }

	    return this.retVal.asObservable();
	  }

	  private onTokenSuccess() {
	    this.container.removeChild(this.spinner);
	    this.label.textContent = this.verified;
	    this.checkbox.disabled = true;
	    this.checkbox.checked = true;
	    this.label.className = 'disabled';
	    this.captchaEnabledSubject.next(true);
	  }

	  override getToken(): string {
	    return this.token;
	  }

	  override ngOnDestroy(): void {
	    if (this.translationSubscription) {
	      this.translationSubscription.unsubscribe();
	    }
	  }

	  private verificationExpired() {
	    this.translationService
	      .translate('errors.captcha.verificationExpired')
	      .pipe(
		switchMap((message: string) => {
		  this.globalMessageService.add(message, GlobalMessageType.MSG_TYPE_ERROR, this.TIMEOUT_MESSAGE);
		  return this.globalMessageService.get();
		}),
		map((globalMessages: any) =>
		  !Object.values(globalMessages || {}).some((msgs: any) =>
		    Array.isArray(msgs) ? msgs.length > 0 : !!msgs
		  )
		),
		filter((noMessages: boolean) => noMessages),
		take(1)
	      )
	      .subscribe(() => {
		  location.reload();
	      });
	  }
	}
```

**Implementação do Componente**

O componente que utiliza o CAPTCHA pode ser semelhante ao seguinte:

```javascript annotate
	export class CustomComponent implements AfterViewInit {

	  protected service = inject(CustomComponentService);
	  protected captchaService = inject(CustomCaptchaService);

	  form: UntypedFormGroup = this.service.form;
	  isUpdating$: Observable<boolean> = this.service.isUpdating$;
	  captchaEnabled$: Observable<boolean> = this.captchaService.captchaEnabled$;
	  captchaRequired$: Observable<boolean> = this.captchaService.captchaRequired$;

	  onSubmit(): void {
	    this.service.requestEmail();
	  }

	  ngAfterViewInit(): void {
	    this.captchaService.initialize();
	  }

	  onCaptchaConfirmed() {
	    this.form.get('captcha')?.setValue(true);
	  }
	}
```


**(English)**

**Introduction to CAPTCHA in SAP Commerce Cloud**

Before implementing CAPTCHA on the frontend, it is important to understand what CAPTCHA is, how it works, and how it is configured in SAP Commerce Cloud.

You can find a detailed explanation in the documentation below: 

https://github.com/viniciusgugelmin/personal-guide/tree/master/src/SAP%20Commerce%20Cloud/Captcha

**Prerequisites**

Before starting the frontend implementation, make sure the following prerequisites are satisfied:

- Google reCAPTCHA keys have been created
- CAPTCHA is configured in SAP Commerce Cloud
- Public and private keys are available in the Commerce environment
- The required IMPEX configurations were executed

Without these steps, the frontend will not receive the CAPTCHA configuration.

**High Level Flow**

The CAPTCHA flow in Spartacus + SAP Commerce Cloud works as follows:

	Spartacus Page
	     ↓
	<cx-captcha> component
	     ↓
	CustomCaptchaService
	     ↓
	Google reCAPTCHA API
	     ↓
	Token generated
	     ↓
	Token sent in request header
	sap-commerce-cloud-captcha-token
	     ↓
	OCC endpoint
	     ↓
	CaptchaValidationInterceptor
	     ↓
	CaptchaValidationService

Once validated successfully, the request is processed normally by SAP Commerce Cloud.

**Getting Started**

To begin the CAPTCHA implementation on the frontend side, it is only necessary to ensure that the following IMPEX files have been executed.

These configurations ensure that:
- the site exposes the CAPTCHA keys to the frontend
- the frontend can send the CAPTCHA token to OCC endpoints without CORS errors

	INSERT_UPDATE BaseStore; uid[unique=true]; captchaCheckEnabled
	; $storeUid ; true

	INSERT_UPDATE CorsConfigurationProperty; context[unique = true]; key[unique = true]; value
	; commercewebservices ; allowedHeaders ; origin content-type accept authorization cache-control if-none-match x-anonymous-consents x-profile-tag-debug x-consent-reference occ-personalization-id occ-personalization-time Sap-Commerce-Cloud-User-Id sap-commerce-cloud-captcha-token

**Verifying CAPTCHA Configuration**

After running the IMPEX files, open the homepage and inspect the BaseSite request.

The response should include the following structure:

	{
	  "...": "...",
	  "captchaConfig": {
	    "enabled": true,
	    "publicKey": "<PUBLIC_KEY>"
	  }
	}

This configuration becomes available inside the Spartacus CaptchaService.

**Starting the Frontend Implementation**

To implement CAPTCHA on the frontend, it is necessary to create a custom service that extends the Spartacus CaptchaService.

This is required because CaptchaService in Spartacus is an abstract class, meaning it cannot be instantiated directly and must be extended.

Example:

	CustomCaptchaService extends CaptchaService

**Registering the Custom CAPTCHA Service**

Before implementing the CAPTCHA behavior, it is necessary to register the custom service in the module where CAPTCHA will be used.

Inside the module providers:

```javascript annotate
	provideConfig(<CaptchaApiConfig>{
	  captchaRenderer: CustomCaptchaService,
	})
```
	
This tells Spartacus to use the custom implementation instead of the default one.

**Adding CAPTCHA to a Component**

After registering the service, CAPTCHA can be rendered inside any component template.

Example:

	<cx-captcha (confirmed)="onCaptchaConfirmed()"></cx-captcha>
	
This component will trigger the CAPTCHA flow and notify the component when the validation succeeds.

**Customizing the CAPTCHA Behavior**

To create a simple but user-friendly CAPTCHA experience, some methods from the CaptchaService must be overridden.

Additionally, some helper logic is introduced to improve:
- user feedback
- token expiration handling
- UI state management

**Adding Translations**

To support internationalization, translations should be added to the Spartacus translation module.

Example:

```javascript annotate
	export const <SPARTACUS_TRANSLATION_MODULE> = {
	  captcha: {
	    succeed: 'Succeed',
	    verified: 'Verified',
	    robot: 'I\'m not a robot',
	  }
	}
```

**Loading Translations in the Service**

Inside the constructor of the CustomCaptchaService, we subscribe to translation values using TranslationService.

This allows dynamic translation updates when the language changes.

```javascript annotate
	this.translationSubscription = combineLatest([
	  this.translationService.translate('captcha.succeed'),
	  this.translationService.translate('captcha.verified'),
	  this.translationService.translate('captcha.robot')
	]).pipe(
	  map(([succeed, verified, robot]) => {
	    this.succeed = succeed;
	    this.verified = verified;
	    this.robot = robot;
	  })
	).subscribe();
```

For this implementation, make sure to inject TranslationService and declare the variables:
- succeed
- verified
- robot

**Overriding the initialize Method**

The initialize() method is responsible for loading the CAPTCHA configuration and preparing the UI.

Here we use the built-in Spartacus method:

```javascript annotate
	fetchCaptchaConfigFromServer()
```

This retrieves the configuration from SAP Commerce.

```javascript annotate
	override initialize() {
	  this.subscription.add(
	    this.fetchCaptchaConfigFromServer().subscribe((config) => {
	      if (config?.enabled) {
		this.captchaConfig = config;
		this.loadResource();
	      } else {
		this.captchaConfigSubject$.next({ enabled: false });
		this.captchaRequired$.next(false);
	      }
	    })
	  );
	}
```

This logic ensures that:
- CAPTCHA renders only when enabled in Commerce
- it remains disabled otherwise

The variable captchaRequired$ should be defined as:

```javascript annotate
	BehaviorSubject<boolean>
```

This controls whether CAPTCHA validation is required.

**Rendering the CAPTCHA UI**

Inside the same initialize() method, we dynamically create a checkbox-based UI.

This checkbox simulates a simple "I'm not a robot" interaction.

```javascript annotate
	this.container = document.createElement('div');
	this.container.className = 'form-check';

	this.checkbox = document.createElement('input');
	this.checkbox.type = 'checkbox';
	this.checkbox.id = 'custom-recaptcha-checkbox';

	this.label = document.createElement('label');
	this.label.textContent = this.robot;
	this.label.htmlFor = this.checkbox.id;

	this.container.appendChild(this.checkbox);
	this.container.appendChild(this.label);

	this.spinner = document.createElement('icon');
	this.spinner.className = 'fa-solid fa-spinner';

	this.checkbox.addEventListener('change', this.onCheckBoxClicked.bind(this));
```

New variables introduced:
- container
- checkbox
- label
- spinner

These are used to build the dynamic UI.

**Handling the CAPTCHA Click**

The onCheckBoxClicked() method triggers the CAPTCHA validation.

This method:
- loads Google reCAPTCHA
- disables the checkbox
- shows a spinner
- starts token generation

```javascript annotate
	onCheckBoxClicked(): void {
	  this.grecaptcha();
	  this.label.textContent = '';
	  this.container.appendChild(this.spinner);
	  this.checkbox.disabled = true;
	  this.checkbox.checked = true;
	  this.label.className = 'disabled';
	  this.retVal.next(this.succeed);
	  this.retVal.complete();
	}
```

**Overriding renderCaptcha**

The renderCaptcha() method is responsible for attaching the CAPTCHA UI to the DOM.

```javascript annotate
	override renderCaptcha(renderParams: RenderParams): Observable<string> {
	  if (renderParams.element instanceof HTMLElement) {
	    this.checkbox.disabled = false;
	    this.checkbox.checked = false;
	    this.label.textContent = this.robot;
	    this.label.className = '';
	    this.retVal = new Subject<string>();

	    renderParams.element.appendChild(this.container);
	  }

	  return this.retVal.asObservable();
	}
```

This ensures that every time CAPTCHA is rendered:
- the UI resets
- the checkbox is enabled
- a new observable response is created

**Required Additional Overrides**

Some additional methods must be overridden to support the custom behavior.

Returns the generated CAPTCHA token.

```javascript annotate
	override getToken(): string {
	  return this.token;
	}
```

Ensures subscriptions are cleaned up to prevent memory leaks.

```javascript annotate
	override ngOnDestroy(): void {
	  if (this.translationSubscription) {
	    this.translationSubscription.unsubscribe();
	  }
	}
```

**Handling CAPTCHA Token Expiration**

Google reCAPTCHA tokens expire after a short period of time.

If the user validates CAPTCHA but delays submitting the form, the token may become invalid.

To improve user experience, a custom handler was implemented:
- display an error message
- notify the user
- reload the page to restart the process

This logic is implemented in:

```javascript annotate
	verificationExpired()
```

**Custom CAPTCHA Service**

```javascript annotate
	@Injectable({
	  providedIn: 'root'
	})
	export class CustomCaptchaService extends CaptchaService implements OnDestroy {
	  protected translationService: TranslationService = inject(TranslationService);
	  protected globalMessageService = inject(GlobalMessageService);

	  protected retVal: Subject<string> = new Subject<string>();
	  protected captchaEnabledSubject: BehaviorSubject<boolean> = new BehaviorSubject<boolean>(false);
	  public captchaEnabled$ = this.captchaEnabledSubject.asObservable();
	  public captchaRequired$ = new BehaviorSubject<boolean>(true);
	  protected container!: HTMLDivElement;
	  protected checkbox!: HTMLInputElement;
	  protected label!: HTMLLabelElement;
	  protected spinner!: HTMLElement;

	  private readonly TIMEOUT_MESSAGE = 5000;
	  private readonly TOKEN_EXPIRED = 120000;

	  private succeed = '';
	  private verified = '';
	  private robot = '';
	  private translationSubscription: any;

	  constructor() {
	    super(
	      inject(SiteAdapter),
	      inject(CaptchaApiConfig),
	      inject(LanguageService),
	      inject(ScriptLoader),
	      inject(BaseSiteService)
	    );

	    this.translationSubscription = combineLatest([
	      this.translationService.translate('captcha.succeed'),
	      this.translationService.translate('captcha.verified'),
	      this.translationService.translate('captcha.robot')
	    ]).pipe(
	      map(([succeed, verified, robot]) => {
		this.succeed = succeed;
		this.verified = verified;
		this.robot = robot;
	      })
	    ).subscribe();
	  }

	  override initialize() {
	    this.subscription.add(
	      this.fetchCaptchaConfigFromServer().subscribe((config) => {
		if (config?.enabled) {
		  this.captchaConfig = config;
		  this.loadResource();
		} else {
		  this.captchaConfigSubject$.next({ enabled: false });
		  this.captchaRequired$.next(false);
		}
	      })
	    );

	    this.container = document.createElement('div');
	    this.container.className = 'form-check';

	    this.checkbox = document.createElement('input');
	    this.checkbox.type = 'checkbox';
	    this.checkbox.id = 'custom-recaptcha-checkbox';

	    this.label = document.createElement('label');
	    this.label.textContent = this.robot;
	    this.label.htmlFor = this.checkbox.id;
	    this.container.appendChild(this.checkbox);
	    this.container.appendChild(this.label);

	    this.spinner = document.createElement('icon');
	    this.spinner.className = 'fa-solid fa-spinner';

	    this.checkbox.addEventListener('change', this.onCheckBoxClicked.bind(this));
	  }

	  onCheckBoxClicked(): void {
	    this.grecaptcha();
	    this.label.textContent = '';
	    this.container.appendChild(this.spinner);
	    this.checkbox.disabled = true;
	    this.checkbox.checked = true;
	    this.label.className = 'disabled';
	    this.retVal.next(this.succeed);
	    this.retVal.complete();
	  }

	  private grecaptcha() {
	    const publicKey = this.captchaConfig.publicKey;

	    this.scriptLoader.embedScript({
	      src: 'https://www.google.com/recaptcha/api.js?render=' + publicKey
	    });

	    const checkGrecaptcha = () => {
	      // @ts-ignore
	      if (typeof grecaptcha !== 'undefined' && grecaptcha.ready) {
		// @ts-ignore
		grecaptcha.ready(() => {
		  // @ts-ignore
		  grecaptcha.execute(publicKey, { action: 'submit' }).then((token: string) => {
		    this.token = token;
		    this.onTokenSuccess();
		    setTimeout(() => {
		      this.verificationExpired();
		    }, this.TOKEN_EXPIRED);
		  });
		});
	      } else {
		setTimeout(checkGrecaptcha, 100);
	      }
	    };

	    checkGrecaptcha();
	  }

	  override renderCaptcha(renderParams: RenderParams): Observable<string> {
	    if (renderParams.element instanceof HTMLElement) {
	      this.checkbox.disabled = false;
	      this.checkbox.checked = false;
	      this.label.textContent = this.robot;
	      this.label.className = '';
	      this.retVal = new Subject<string>();

	      renderParams.element.appendChild(this.container);
	    }

	    return this.retVal.asObservable();
	  }

	  private onTokenSuccess() {
	    this.container.removeChild(this.spinner);
	    this.label.textContent = this.verified;
	    this.checkbox.disabled = true;
	    this.checkbox.checked = true;
	    this.label.className = 'disabled';
	    this.captchaEnabledSubject.next(true);
	  }

	  override getToken(): string {
	    return this.token;
	  }

	  override ngOnDestroy(): void {
	    if (this.translationSubscription) {
	      this.translationSubscription.unsubscribe();
	    }
	  }

	  private verificationExpired() {
	    this.translationService
	      .translate('errors.captcha.verificationExpired')
	      .pipe(
		switchMap((message: string) => {
		  this.globalMessageService.add(message, GlobalMessageType.MSG_TYPE_ERROR, this.TIMEOUT_MESSAGE);
		  return this.globalMessageService.get();
		}),
		map((globalMessages: any) =>
		  !Object.values(globalMessages || {}).some((msgs: any) =>
		    Array.isArray(msgs) ? msgs.length > 0 : !!msgs
		  )
		),
		filter((noMessages: boolean) => noMessages),
		take(1)
	      )
	      .subscribe(() => {
		  location.reload();
	      });
	  }
	}
```

**Component Implementation**

The component using CAPTCHA may look like the following:

```javascript annotate
	export class CustomComponent implements AfterViewInit {

	  protected service = inject(CustomComponentService);
	  protected captchaService = inject(CustomCaptchaService);

	  form: UntypedFormGroup = this.service.form;
	  isUpdating$: Observable<boolean> = this.service.isUpdating$;
	  captchaEnabled$: Observable<boolean> = this.captchaService.captchaEnabled$;
	  captchaRequired$: Observable<boolean> = this.captchaService.captchaRequired$;

	  onSubmit(): void {
	    this.service.requestEmail();
	  }

	  ngAfterViewInit(): void {
	    this.captchaService.initialize();
	  }

	  onCaptchaConfirmed() {
	    this.form.get('captcha')?.setValue(true);
	  }
	}
```

**SOURCE**
---

SAP. CAPTCHA. Available at: https://help.sap.com/docs/SAP_COMMERCE_COMPOSABLE_STOREFRONT/eaef8c61b6d9477daf75bff9ac1b7eb4/4605061113284c2c92cc558600140aaa.html. Accessed on: Mar. 10, 2026.

