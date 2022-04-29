
#  Integração IBEU Site

Toda integração e manipulação dos dados do formulário (Prospecção e Contatos) estão sendo feitas diretamente no PHP, sendo requisições feita SSR (Server Side Rendering) para o front-end.

  
<br />
<br />

##  Eventos Exportados

na pasta `/wp-content/themes/hello-elementor/includes/customizer/` existe um arquivo chamado `ibeu-integrations.php` que exporta os eventos do IBEU para o WordPress. São eles:

<br />

### `GetPageUnities`
Essa função tem como objetivo identificar qual URL o usuário está acessando para obter uma filial/código de produto específico e assim manipular esse dado para uma possível alteração no formulário.

    public function getPageUnities($page) {
		
		// Valor Padrão do parâmetro.
	    $params  =  array(
		    'filial'  =>  1,
		    'codProduto'  =>  1
	    );
	    
	    // Mudança dos parâmetro de acordo com a URL.
	    switch  ($page)  {
		    case  '/cursos-de-ingles/high-school/':
			    $params['codProduto']  =  6;
			    break;
    
			case  '/exames-internacionais/':
			    $params['filial']  =  "99M";
			    $params['codProduto']  =  9;
			    break;
			    
		    case  '/preparatorios-para-exames/':
			    $params  =  false;
			    break;
    
		    case  '/ingles-nas-escolas/':
			    $params['filial']  =  "99M";
			    $params['codProduto']  =  12;
			    break;
    
		    case  '/ingles-para-empresas/':
			    $params['filial']  =  56;
			    $params['codProduto']  =  1;
			    break;
	    }
	    
	    // Retorno do parâmetro
	    return  $params;
    }
   
   #### parâmetros da função
   
| nome | tipo | exemplo |
|--|--|--|
| page | `string` | /ingles-para-empresas/ |


<br />
<br />

### `fetch`
Essa função tem como objetivo fazer requisições para api da IBEU, ela trata a requisição e retorna os valores para serem utilizados na integração.


    public function fetch($url, $method = "GET", $args = array())
    {
	    //Caso a variável `$args` for vazia, irá pegar o valor padrão
	    if (empty($args)) {
		    $args  =  $this->base_args; 
	    }  
    
	    // Tipo de requisição "GET"
	    if  ($method == 'GET') {
		    $response = wp_remote_get($this->base_url . $url, $args);  

	    // Tipo de requisição "POST"  
	    } elseif ($method == 'POST') {
		    $response = wp_remote_post($this->base_url . $url, $args);
		   
	    // Caso não seja nem "GET" ou "POST"
	    } else {
			return new WP_Error('unsupported', "Tipo de requisição inválida");
	    }
		
		// Tratando o corpo da requisição para retornar os dados
	    if (!($body = json_decode($response['body']))) {
		    $body = $response['body'];
	    }
    
	    // Caso ocorra algum erro na requisição
	    if (!empty($body->error)) {
		    $message = $body->error . ' (Status Code: ' . $response['code'] . ')';
		    throw  new  WP_Error($message);
	    }
	    
	    // Caso ocorra algum erro inesperado
	    if (is_wp_error($response)) {
			$message = $response->get_error_message();    
		    throw new WP_Error("ERROR");
	    }
	    
	    // Tratando a resposta da requisição
	    $results = array(
		    'body' => $body,
		    'code' => $response['response']['code'],
		    'headers' => $response['headers']
		);
	    
	    // Retornando o resultado
	    return  $results;
    }

 ### parâmetros da função
| nome | tipo | exemplo |
|--|--|--|
| url | `string` | /read?action=areaContato |
| method | `string` | POST |
| args | `array` | array("body" => array("email" => "teste@teste.com") |

<br />
<br />

##  Função de gatilhos (wordpress)
Agora com as funções criadas e exportadas para uso dentro do wordpress precisamos que essas funções sejam chamadas em gatilhos para assim buscar os dados e exibir para usuário.

<br />

Na pasta `/wp-content/themes/hello-elementor/includes/` existe um arquivo chamado `customizer-functions-ibeu.php` que importa os eventos listados em [Eventos Exportados](#eventos-exportados) e ativa os gatilhos necessários, essas funções em gatilhos são:

<br />

### `integration_ibeu_api_unities`
Essa função tem como objetivo capturar a URL do usuário, pegar o código de produto/filial através da função de evento [`getPageUnities`](#getpageunities) fazer a requisição com a função de evento [`fetch`](#fetch), com os dados retornados para exibir as unidades no select box do formulário.

<br />

> obs: Essa função só é disparada caso haja um selectbox chamado filial, ou seja, apenas as páginas que realmente precisam desse tipo de consulta de unidades.

<br />

    function integration_ibeu_api_unities($scanned_tag) {
	    try {
		    // Verifica se o nome do campo selectbox existe
		    if ($scanned_tag['name'] != "filial") :
			    return $scanned_tag;
		    endif;
		    
		    // Chama os eventos criados
		    $ibeu = new Ibeu\Includes\Integration\Events;
		    
		    // Captura a URL do usuário
		    $RequestURI = filter_input(INPUT_SERVER, 'REQUEST_URI');
			
			// Chama o evento getPageUnities
			$params = $ibeu->getPageUnities($RequestURI); 
		    
		    // Chama o evento fetch para pegar os dados das unidades
		    $response = $ibeu->fetch('/read?action=campus' . ($params ? '&' . http_build_query($params) : ''));
	
			// Retorno caso ocorra algum erro na requisição    
		    if (!$response['body'] || $response['body']->StatusCode !=  200) {    
			    return;
		    }
			
			// Capturando os dados das unidades
		    $datas = $response['body']->Data->Campus;
		    
		    // Populando o selectbox do formulário com as unidades retornadas
		    foreach  ($datas  as  $data)  {
			    foreach  ($data  as  $item)  {
				    $scanned_tag['raw_values'][] = $item->descricao . "|" . $item->codCampus;
    			}
		    }
    
	      
		    // Verificando o tipo de plugin do fomulário no Wordpress
		    if  (WPCF7_USE_PIPE)  {
		    
			    // fazendo o update no formulário depois de inserir os dados das unidades
			    $pipes  =  new  WPCF7_Pipes($scanned_tag['raw_values']);
			    $scanned_tag['values']  =  $pipes->collect_afters();
			    $scanned_tag['labels']  =  $pipes->collect_befores();
			    $scanned_tag['pipes']  =  $pipes;
		    }  else  {
		    
			    // fazendo o update no formulário depois de inserir os dados das unidades
			    $scanned_tag['values']  =  $scanned_tag['raw_values'];
			    $scanned_tag['labels']  =  $scanned_tag['values'];
		    }
    
      
		    // Retornando o formulário atualizado para o usuário
		    return  $scanned_tag;
	    }  catch  (Exception  $e)  {
	    
		    // Retorno caso haja um erro inesperado
		    print_r($e->getMessage());
	    }
    }


<br />
<br />

### `integration_ibeu_api_contact`
Essa função tem como objetivo fazer a requisição para API para retornar os dados de assunto do formulário de contato que fica em `/fale-conosco/`.

<br />

> obs: Essa função só é disparada quando a URL é /fale-conosco/ e se houver um selectbox chamado "assunto".

<br />

    function  integration_ibeu_api_contact($scanned_tag) {
	    try  {
		   // Verifica se o nome do campo selectbox existe
		   if  ($scanned_tag['name']  !=  "assunto")  :
			   return  $scanned_tag;
		   endif;

		   // Chama os eventos criados
		   $ibeu = new Ibeu\Includes\Integration\Events;
		    
		   // Captura a URL do usuário
		   $RequestURI = filter_input(INPUT_SERVER, 'REQUEST_URI');
		    
		   // Verificando a URL "/fale-conosco/"
		   if ($RequestURI  ===  '/fale-conosco/') :
			   $response  =  $ibeu->fetch('/read?action=areaContato');
			
				// Retorno caso ocorra algum erro na requisição
			    if  (!$response['body']  ||  $response['body']->StatusCode  !=  200)  {
				    return;
				}
			
			    // Capturando os dados das unidades
			    $datas  =  $response['body']->Data->AreaContato;
		    
				// Populando o selectbox do formulário com os assuntos retornados
				foreach  ($datas  as  $data)  {
					foreach  ($data  as  $item)  {
						$scanned_tag['raw_values'][]  =  $item->nome  .  "|"  .  $item->codAreaContato;
					}
				}
			
			    // Verificando o tipo de plugin do fomulário no Wordpress
			    if  (WPCF7_USE_PIPE)  {
		    
				    // fazendo o update no formulário depois de inserir os dados das unidades
				    $pipes  =  new  WPCF7_Pipes($scanned_tag['raw_values']);
				    $scanned_tag['values']  =  $pipes->collect_afters();
				    $scanned_tag['labels']  =  $pipes->collect_befores();
				    $scanned_tag['pipes']  =  $pipes;
			    }  else  {
		    
				    // fazendo o update no formulário depois de inserir os dados das unidades
				    $scanned_tag['values']  =  $scanned_tag['raw_values'];
				    $scanned_tag['labels']  =  $scanned_tag['values'];
			    }

			    // Retornando o formulário atualizado para o usuário
			    return  $scanned_tag;
			endif;
		}  catch  (Exception  $e)  {
	    
		    // Retorno caso haja um erro inesperado
		    return  $e->getMessage();
		}
	}
	
<br />
<br />

### `integration_ibeu_api_prospect`
Essa função tem como objetivo enviar os dados do formulário para API.

<br />

> obs: Essa função só é disparada quando o formulário de prospecção for validádo e enviado pelo usuário.

<br />

    function integration_ibeu_api_prospect($cf7,  $abort) {
	    try  {
			//Função nativa do Wordpress para pegar informações do formulário.
		    $wpcf = WPCF7_ContactForm::get_current(); // informações básicas
		    $submission = WPCF7_Submission::get_instance(); // Submit do formulário
		    $form_id = $wpcf->id(); //Pegando o ID do formulário
			   
			//Verificação dos IDS de formulário de prospecção
		    if (!($form_id  ==  16474 || $form_id  ==  17403)) :
			    $wpcf->skip_mail  =  true;
			    return  $cf7;
		    endif;
    
		    // Chama os eventos criados
		   $ibeu = new Ibeu\Includes\Integration\Events;
		    
		   // Captura a URL do usuário
		   $RequestURI = filter_input(INPUT_SERVER, 'REQUEST_URI');
		   
		   //Capturar os dados de unidades/filiais
		   $getParamsUnity  =  $ibeu->getPageUnities($RequestURI);
		   
		   //Variável que define se é uma empresa ou não
			$isCompany  =  false;
		    
		    //Captura dos dados dos campos do formulário
		    $email  =  $submission->get_posted_data('email');
		    $nome  =  $submission->get_posted_data('nome');
		    $telefone  =  $submission->get_posted_data('telefone');
		    $receberComunicacoes  =  $submission->get_posted_data('recebe-comunicacoes');
		    $estudaNoIbeu  =  $submission->get_posted_data('estuda-no-ibeu[]');
		    $filial  =  $submission->get_posted_data('filial');
		    
		    //Verificando se é uma empresa
		    if  ($RequestURI  ===  '/ingles-nas-escolas/'  ||  $RequestURI  ===  '/ingles-para-empresas/')  {
			    $isCompany  =  true;
		    }
	    
		    //Montando o cabeçalho e o corpo da requisição
			$args  =  array(
			    'headers'  =>  array(
				    'login'  =>  'WebAPI.Area51.User',
				    'password'  =>  '84@OO-$HKvCaM',
				    'ambiente'  =>  'Treinamento',
			    ),
			    'body'  =>  array(
				    'email'  =>  $email,
				    'nome'  =>  $nome,
				    'dataNascimento'  =>  "",
				    'nomeEmpresa'  =>  $isCompany  ?  $nome  :  "",
				    'telefone3'  =>  $telefone,
				    'concordaReceberContato'  =>  $receberComunicacoes  ==  "SIM"  ?  1  :  0,
				    'codProduto'  =>  $getParamsUnity['codProduto'],
				    'codOrigem'  =>  4,
				    'estudaNoIBEU'  =>  $estudaNoIbeu  ?  1  :  0,
				    'pessoaFisOUJur'  =>  $isCompany  ?  "J"  :  "F",
				    'codCampus'  =>  $isCompany  ?  $getParamsUnity['filial']  :  ($filial  ===  'ESCOLHA A FILIAL'  ||  $filial  ===  'CENTRAL DE ATENDIMENTO')  ?  "99"  :  $filial[0],
			    )
		    );
		    
		    //Enviando requisição para api
		    $response  =  $ibeu->fetch('/create?action=prospectIBEU',  "POST",  $args);
			
			//Caso haja algum erro na requisição abortar o envio do formulário
		    if  (!$response['body']  ||  $response['body']->StatusCode  !=  200)  {
			    $abort  =  true;
		    }
		    
		    //Atualizando os status do formulário
		    $wpcf->skip_mail  =  true;
		    return  $cf7;
    
		} catch (Exception $e) {
	    
		    // Retorno caso haja um erro inesperado
		    print_r($e->getMessage());
		}
	}

## Formulário

Se você quiser reutilizar as integrações do selectbox feito no formulário de proscrição ou no formulário de fale conosco, você pode fazer isso aqui.

> Entre no wordpress, no menu lateral, clique em **Contato -> Adicionar novo**

- Puxar a integração das filiais você precisa criar um select passando o nome `filial` e o `id:filial` como no exemplo abaixo:

```
[select filial id:filial "ESCOLHA A FILIAL" "CENTRAL DE ATENDIMENTO|99"]
```

- Puxar a integração do assunto no formulário "Fale Conosco" você precisa criar um select passando o nome `assunto` e o `id:assunto` como no exemplo abaixo:

```
[select assunto id:assunto "Escolha um assunto"]
```
Caso queira adicionar uma nova integração como o formulário de prospecção porém passando uma filial fixa como em algumas páginas, você pode fazer isso aqui.

- Primeiro você precisa puxar o formulário de prospecção para a página ou criar um novo formulário com os mesmos campos.

- Você ira acessar o arquivo `ibeu-integrations.php` vá até a função `getPageUnities` e adiciona um novo `case`, exemplo:

```
// Mudança dos parâmetro de acordo com a URL.
switch ($page) {
    case  7503: // /cursos-de-ingles/high-school/
        $params['codProduto']  =  6;
        break;

    case  5972: // /exames-internacionais/
        $params['filial']  =  "99";
        $params['codProduto']  =  9;
        break;

    case  5201: // /preparatorios-para-exames/
        $params['codProduto']  =  1;
        break;

    case  1556: // /ingles-nas-escolas/
        $params['filial']  =  "99M";
        $params['codProduto']  =  12;
        break;

    case  7018: // /ingles-para-empresas/
        $params['filial']  =  56;
        $params['codProduto']  =  1;
        break;

    case ID_DA_PÁGINA: // /Nova página
        $params['filial']  =  NUMERO_DA_FILIAL;
        break;
}
```

Depois de adicionado o novo `case`, você precisa adicionar o `ID DA PÁGINA` na hora de submeter o novo formulário, pra isso você agora vai no arquivo `customizer-functions-ibeu.php` proucure a função `integration_ibeu_api_prospect` na linha 153, lendo a função você vai achar uma variável chamada `isFixedParams` ela serve justamente para identificar a página que tem os parâmetros fixos de filias.

Na linha 196 você vai encontrar um `if`, ele identifica quais páginas tem os parâmetros fixos de filiais e seta isso nos parâmetros na hora do envio do formulário, você irá altera-lô, que ficara assim:

```
if ($RequestURI === 7702 || $RequestURI === ID_DA_NOVA_PAGINA) {
    $isFixedParams  =  true;
}
```

Feito, se acessar agora a página com o novo formulário e submeter ele você irar percebe que os valores vão ser enviados corretamente.
