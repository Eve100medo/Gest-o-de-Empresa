sap.ui.define([
    "sap/ui/core/mvc/Controller",
    "sap/ui/core/Fragment",
    "sap/m/MessageBox",
    "sap/m/MessageToast",
    "sap/ui/model/json/JSONModel",
    "sap/ui/model/Filter",
    "sap/ui/model/FilterOperator"
],
    function (Controller,
        Fragment,
        MessageBox,
        MessageToast,
        JSONModel,
        Filter,
        FilterOperator
    ) {
        "use strict";

        return Controller.extend("com.ysui5petrobras.br.ysui5.petrobras.freestyle.controller.View1", {

            // Variável de controle interna para evitar cliques simultâneos
            _bEstaSalvando: false,

            onInit: function () {

                // Modelo local do Dialog para controlar textos, visibilidade e inputs
                var oDialogModel = new JSONModel({
                    contextoAtual: "",
                    entitySet: "",
                    sPath: "",
                    modo: "",
                    tituloDialog: "",
                    labelChave: "",
                    labelNome: "",
                    chave: "",
                    nome: "",
                    atribCNPJ: "",
                    atribCPF: "",
                    nomeEmpresaAtrib: "",
                    nomeInspetorAtrib: "",
                    dataInicio: null,
                    dataFim: null,
                    chaveEditavel: true,
                    exibirChaveGeral: true,
                    exibirAtribuicao: false
                });
                this.getView().setModel(oDialogModel, "modelDialog");

                // Configuração para carregamento automático inicial (Ignorar necessidade do botão GO)
                this._setupSmartTablesAutoLoad();

            },
            /**
                * Força as SmartTables a realizarem o fetch/carregamento de dados de forma automática
                */
            _setupSmartTablesAutoLoad: function () {
                var aSmartTableIds = ["tblEmpresa", "tblInspetor", "tblAtribuicao"];

                aSmartTableIds.forEach(function (sId) {
                    var oSmartTable = this.byId(sId);
                    if (oSmartTable) {
                        // Define a propriedade para inicializar os dados sem depender do clique no botão GO
                        oSmartTable.setEnableAutoBinding(true);

                        // Garante re-disparo do binding caso o componente já estivesse renderizado em cache
                        if (oSmartTable.isInitialised()) {
                            oSmartTable.rebindTable();
                        } else {
                            oSmartTable.attachInitialise(function () {
                                oSmartTable.rebindTable();
                            });
                        }
                    }
                }.bind(this));
            },

            _getODataModel: function () {
                return this.getView().getModel();
            },

            /**
            * Gerenciador central do Dialog Principal (Criar / Editar)
            */
            _openDialog: function (sContexto, sModo, sSmartTableId) {
                var oSmartTable = this.byId(sSmartTableId);
                var oTable = oSmartTable.getTable();
                var aSelectedItems = oTable.getSelectedItems();
                var oDialogModel = this.getView().getModel("modelDialog");

                if (sModo === "EDITAR" && aSelectedItems.length === 0) {
                    MessageBox.error("Por favor, selecione uma linha na tabela para editar.");
                    return;
                }

                var oConfig = {};
                if (sContexto === "EMPRESA") {
                    oConfig = { entitySet: "YDE_CRUD_CV_EMPRESA", labelChave: "Cnpj", labelNome: "Nome da Empresa", campoChave: "Cnpj", campoNome: "Empresa", titulo: "Empresa" };
                } else if (sContexto === "INSPETOR") {
                    oConfig = { entitySet: "YDE_CRUD_CV_INSPETOR", labelChave: "Cpf", labelNome: "Nome do Inspetor", campoChave: "Cpf", campoNome: "Nome", titulo: "Inspetor" };
                } else if (sContexto === "ATRIBUICAO") {
                    oConfig = { entitySet: "YDE_CRUD_CV_ATRIBUICAO", titulo: "Atribuição" };
                }

                oDialogModel.setProperty("/contextoAtual", sContexto);
                oDialogModel.setProperty("/entitySet", oConfig.entitySet);
                oDialogModel.setProperty("/modo", sModo);
                oDialogModel.setProperty("/tituloDialog", sModo === "CRIAR" ? "Criar " + oConfig.titulo : "Editar " + oConfig.titulo);
                oDialogModel.setProperty("/chaveEditavel", sModo === "CRIAR");
                oDialogModel.setProperty("/exibirChaveGeral", sContexto !== "ATRIBUICAO");
                oDialogModel.setProperty("/exibirAtribuicao", sContexto === "ATRIBUICAO");

                if (sModo === "EDITAR") {
                    var oSelectedItem = aSelectedItems[0];
                    var oContext = oSelectedItem.getBindingContext();
                    var oData = oContext.getObject();

                    oDialogModel.setProperty("/sPath", oContext.getPath());
                    oDialogModel.setProperty("/dataInicio", oData.DataInicio ? new Date(oData.DataInicio) : null);
                    oDialogModel.setProperty("/dataFim", oData.DataFim ? new Date(oData.DataFim) : null);

                    if (sContexto === "ATRIBUICAO") {
                        oDialogModel.setProperty("/atribCNPJ", oData.Cnpj);
                        oDialogModel.setProperty("/atribCPF", oData.Cpf);
                        oDialogModel.setProperty("/nomeEmpresaAtrib", oData.Empresa);
                        oDialogModel.setProperty("/nomeInspetorAtrib", oData.Nome);
                    } else {
                        oDialogModel.setProperty("/labelChave", oConfig.labelChave);
                        oDialogModel.setProperty("/labelNome", oConfig.labelNome);
                        oDialogModel.setProperty("/chave", oData[oConfig.campoChave]);
                        oDialogModel.setProperty("/nome", oData[oConfig.campoNome]);
                    }
                } else {
                    oDialogModel.setProperty("/sPath", "");
                    oDialogModel.setProperty("/chave", "");
                    oDialogModel.setProperty("/nome", "");
                    oDialogModel.setProperty("/labelChave", oConfig.labelChave || "");
                    oDialogModel.setProperty("/labelNome", oConfig.labelNome || "");
                    oDialogModel.setProperty("/atribCNPJ", "");
                    oDialogModel.setProperty("/atribCPF", "");
                    oDialogModel.setProperty("/nomeEmpresaAtrib", "");
                    oDialogModel.setProperty("/nomeInspetorAtrib", "");
                    oDialogModel.setProperty("/dataInicio", null);
                    oDialogModel.setProperty("/dataFim", null);
                }

                var oView = this.getView();
                if (!this._pEditDialog) {
                    this._pEditDialog = Fragment.load({
                        id: oView.getId(),
                        name: "com.ysui5petrobras.br.ysui5.petrobras.freestyle.fragment.DialogEdit",
                        controller: this
                    }).then(function (oDialog) {
                        oView.addDependent(oDialog);
                        return oDialog;
                    });
                }

                this._pEditDialog.then(function (oDialog) {
                    oDialog.open();
                });
            },

            onCriarEmpresa: function () { this._openDialog("EMPRESA", "CRIAR", "tblEmpresa"); },
            onEditarEmpresa: function () { this._openDialog("EMPRESA", "EDITAR", "tblEmpresa"); },
            onDeletarEmpresa: function () { this._onDeletarRegistro("tblEmpresa"); },

            onCriarInspetor: function () { this._openDialog("INSPETOR", "CRIAR", "tblInspetor"); },
            onEditarInspetor: function () { this._openDialog("INSPETOR", "EDITAR", "tblInspetor"); },
            onDeletarInspetor: function () { this._onDeletarRegistro("tblInspetor"); },

            onCriarAtribuicao: function () { this._openDialog("ATRIBUICAO", "CRIAR", "tblAtribuicao"); },
            onEditarAtribuicao: function () { this._openDialog("ATRIBUICAO", "EDITAR", "tblAtribuicao"); },
            onDeletarAtribuicao: function () {
                this._onDeletarRegistro("tblAtribuicao");

            },

            /**
            * Persistência de Dados com Bloqueio de Concorrência via State da Controller
            */
            onSaveDialog: function () {
                if (this._bEstaSalvando) {
                    return;
                }

                var oODataModel = this._getODataModel();
                var oDialogModel = this.getView().getModel("modelDialog");

                var sModo = oDialogModel.getProperty("/modo");
                var sContexto = oDialogModel.getProperty("/contextoAtual");
                var sEntitySet = oDialogModel.getProperty("/entitySet");
                var sPath = oDialogModel.getProperty("/sPath");

                var oDataInicio = oDialogModel.getProperty("/dataInicio");
                var oDataFim = oDialogModel.getProperty("/dataFim");

                var oPayload = {
                    DataInicio: oDataInicio ? new Date(oDataInicio) : null,
                    DataFim: oDataFim ? new Date(oDataFim) : null
                };

                var aFiltrosDuplicidade = [];
                var sMensagemAviso = "";

                if (sContexto === "ATRIBUICAO") {
                    var sCnpjAtrib = oDialogModel.getProperty("/atribCNPJ") || "";
                    var sCpfAtrib = oDialogModel.getProperty("/atribCPF") || "";

                    oPayload.Cnpj = sCnpjAtrib.toString().replace(/[^\d]/g, "");
                    oPayload.Cpf = sCpfAtrib.toString().replace(/[^\d]/g, "");
                    oPayload.Empresa = oDialogModel.getProperty("/nomeEmpresaAtrib") || "";
                    oPayload.Nome = oDialogModel.getProperty("/nomeInspetorAtrib") || "";

                    if (!oPayload.Cnpj || !oPayload.Cpf) {
                        MessageBox.error("CNPJ e CPF são campos obrigatórios para a atribuição.");
                        return;
                    }

                    aFiltrosDuplicidade = [
                        new Filter("Cnpj", FilterOperator.EQ, oPayload.Cnpj),
                        new Filter("Cpf", FilterOperator.EQ, oPayload.CPF)
                    ];
                    sMensagemAviso = "Aviso: Já existe uma atribuição cadastrada no sistema para esta mesma Empresa e este mesmo Inspetor.";

                } else {
                    var sChaveCrua = oDialogModel.getProperty("/chave") || "";
                    var sNome = oDialogModel.getProperty("/nome") || "";
                    var sChaveLimpa = sChaveCrua.toString().replace(/[^\d]/g, "");

                    if (!sChaveLimpa) {
                        MessageBox.error("O preenchimento do identificador (CNPJ/CPF) é obrigatório.");
                        return;
                    }

                    if (sContexto === "EMPRESA") {
                        oPayload.Cnpj = sChaveLimpa;
                        oPayload.Empresa = sNome;

                        aFiltrosDuplicidade = [new Filter("Cnpj", FilterOperator.EQ, oPayload.Cnpj)];
                        sMensagemAviso = "Aviso: Já existe uma Empresa cadastrada com este mesmo CNPJ.";

                    } else if (sContexto === "INSPETOR") {
                        oPayload.Cpf = sChaveLimpa;
                        oPayload.Nome = sNome;

                        aFiltrosDuplicidade = [new Filter("Cpf", FilterOperator.EQ, oPayload.Cpf)];
                        sMensagemAviso = "Aviso: Já existe um Inspetor cadastrado com este mesmo CPF.";
                    }
                }

                this._bEstaSalvando = true;
                this.getView().setBusy(true);

                if (sModo === "CRIAR") {
                    oODataModel.read("/" + sEntitySet, {
                        filters: aFiltrosDuplicidade,
                        success: function (oData) {
                            if (oData && oData.results && oData.results.length > 0) {
                                this.getView().setBusy(false);
                                MessageBox.warning(sMensagemAviso);
                                this._bEstaSalvando = false;
                            } else {
                                this._executeSave(sModo, sPath, sEntitySet, oPayload);
                            }
                        }.bind(this),
                        error: function () {
                            this.getView().setBusy(false);
                            MessageBox.error("Erro técnico ao validar a consistência dos dados cadastrais.");
                            this._bEstaSalvando = false;
                        }.bind(this)
                    });
                    return;
                }

                this._executeSave(sModo, sPath, sEntitySet, oPayload);
            },

            _executeSave: function (sModo, sPath, sEntitySet, oPayload) {
                var oODataModel = this._getODataModel();

                if (sModo === "EDITAR") {
                    oODataModel.update(sPath, oPayload, {
                        success: function () {
                            this.getView().setBusy(false);
                            MessageToast.show("Registro atualizado!");
                            oODataModel.refresh(true);
                            this._bEstaSalvando = false;
                            this.onCloseDialog();
                        }.bind(this),
                        error: function () {
                            this.getView().setBusy(false);
                            MessageBox.error("Falha ao salvar as modificações.");
                            this._bEstaSalvando = false;
                        }.bind(this)
                    });
                } else {
                    oODataModel.create("/" + sEntitySet, oPayload, {
                        success: function () {
                            this.getView().setBusy(false);
                            MessageToast.show("Registro inserido com sucesso!");
                            this._bEstaSalvando = false;
                            this.onCloseDialog();
                        }.bind(this),
                        error: function (oError) {
                            this.getView().setBusy(false);
                            this._bEstaSalvando = false;

                            var sMsg = "Erro ao gravar dados técnicos. Verifique duplicidade de chaves.";
                            try {
                                var oResponse = JSON.parse(oError.responseText);
                                if (oResponse.error && oResponse.error.message) {
                                    sMsg = oResponse.error.message.value;
                                }
                            } catch (e) { }
                            MessageBox.error(sMsg);


                            //  LIMPAR CAMPOS SE FOR ERRO DE VÍNCULO
                            if (sMsg && sMsg.includes("vínculo ativo") || sMsg.includes("CPF informado é inválido") || sMsg.includes("CNPJ informado é inválido")) {

                                var oDialogModel = this.getView().getModel("modelDialog");

                                oDialogModel.setProperty("/atribCPF", "");
                                oDialogModel.setProperty("/atribCNPJ", "");
                                oDialogModel.setProperty("/chave", "");
                                oDialogModel.setProperty("/nome", "");
                                oDialogModel.setProperty("/nomeInspetorAtrib", "");
                                oDialogModel.setProperty("/nomeEmpresaAtrib", "");




                            }

                        }.bind(this)
                    });
                }
            },

            _onDeletarRegistro: function (sSmartTableId) {
                var oSmartTable = this.byId(sSmartTableId);
                var oTable = oSmartTable.getTable();
                var aSelectedItems = oTable.getSelectedItems();

                if (aSelectedItems.length === 0) {
                    MessageBox.error("Selecione pelo menos uma linha para deletar.");
                    return;
                }

                var oODataModel = this._getODataModel();
                var sMsgConfirmacao = aSelectedItems.length === 1
                    ? "Deseja realmente remover este registro?"
                    : "Deseja realmente remover os " + aSelectedItems.length + " registros selecionados?";

                MessageBox.confirm(sMsgConfirmacao, {
                    actions: [MessageBox.Action.OK, MessageBox.Action.CANCEL],
                    onClose: function (sAction) {
                        if (sAction === MessageBox.Action.OK) {
                            this.getView().setBusy(true);

                            var sActionPath = "";
                            var sParamName = "";
                            var sFieldName = "";

                            if (sSmartTableId === "tblEmpresa") {
                                sActionPath = "/deletarEmpresa";
                                sParamName = "IdEmpresa";
                                sFieldName = "IdEmpresa";
                            } else if (sSmartTableId === "tblInspetor") {
                                sActionPath = "/deletarRegistro";
                                sParamName = "IdInspetor";
                                sFieldName = "IdInspetor";
                            } else if (sSmartTableId === "tblAtribuicao") {
                                sActionPath = "/deletaratribuicao";
                                sParamName = "IdAtribuicao";
                                sFieldName = "IdAtribuicao";
                            }

                            var aPromises = [];

                            aSelectedItems.forEach(function (oItem) {
                                var oContext = oItem.getBindingContext();
                                var oData = oContext.getObject();

                                var oUrlParams = {};
                                oUrlParams[sParamName] = oData[sFieldName];

                                var oCallPromise = new Promise(function (resolve, reject) {
                                    oODataModel.callFunction(sActionPath, {
                                        method: "POST",
                                        urlParameters: oUrlParams,
                                        success: function (oData, response) {
                                            resolve(response);
                                        },
                                        error: function (oError) {
                                            reject(oError);
                                        }
                                    });
                                });

                                aPromises.push(oCallPromise);
                            });

                            Promise.all(aPromises)
                                .then(function () {
                                    this.getView().setBusy(false);
                                    MessageToast.show("Registro(s) processado(s) com sucesso!");
                                    oTable.removeSelections();
                                    oODataModel.refresh(true);
                                }.bind(this))
                                .catch(function (oError) {
                                    this.getView().setBusy(false);
                                    MessageBox.error("Erro ao tentar desativar um ou mais registros.");
                                    oODataModel.refresh(true);
                                }.bind(this));
                        }
                    }.bind(this)
                });
            },

            /**
            * =================================================================
            * LÓGICA DOS BOTÕES DE AJUDA DE PESQUISA (VALUE HELP / F4) 
            * =================================================================
            */

            onValueHelpCNPJ: function () {
                this._sValueHelpTarget = "EMPRESA";
                this._openValueHelp("YDE_CRUD_CV_EMPRESA", "Cnpj", "Empresa", "Buscar Empresa (F4)");
            },

            onValueHelpCPF: function () {
                this._sValueHelpTarget = "INSPETOR";
                this._openValueHelp("YDE_CRUD_CV_INSPETOR", "Cpf", "Nome", "Buscar Inspetor (F4)");
            },

            _openValueHelp: function (sEntitySet, sCampoChave, sCampoTexto, sTitulo) {
                var oView = this.getView();

                if (!this._pValueHelpDialog) {
                    this._pValueHelpDialog = Fragment.load({
                        id: oView.getId(),
                        name: "com.ysui5petrobras.br.ysui5.petrobras.freestyle.fragment.ValueHelpDialog",
                        controller: this
                    }).then(function (oDialog) {
                        oView.addDependent(oDialog);
                        return oDialog;
                    });
                }

                this._pValueHelpDialog.then(function (oDialog) {
                    oDialog.setTitle(sTitulo);

                    // Destrói qualquer resquício de binding anterior antes de injetar novos parâmetros
                    oDialog.unbindAggregation("items");

                    // Força paginação via OData com limite de 20 registros por lote
                    oDialog.bindAggregation("items", {
                        path: "/" + sEntitySet,
                        parameters: {
                            startIndex: 0,
                            length: 20
                        },
                        template: new sap.m.StandardListItem({
                            title: "{" + sCampoChave + "}",
                            description: "{" + sCampoTexto + "}",
                            type: "Active"
                        })
                    });

                    oDialog.open();
                });
            },

            _onValueHelpConfirm: function (oEvent) {
                var oSelectedItem = oEvent.getParameter("selectedItem");
                if (!oSelectedItem) { return; }

                var sChaveEscolhida = oSelectedItem.getTitle();
                var sTextoEscolhido = oSelectedItem.getDescription();
                var oDialogModel = this.getView().getModel("modelDialog");

                if (this._sValueHelpTarget === "EMPRESA") {
                    oDialogModel.setProperty("/atribCNPJ", sChaveEscolhida);
                    oDialogModel.setProperty("/nomeEmpresaAtrib", sTextoEscolhido);
                } else if (this._sValueHelpTarget === "INSPETOR") {
                    oDialogModel.setProperty("/atribCPF", sChaveEscolhida);
                    oDialogModel.setProperty("/nomeInspetorAtrib", sTextoEscolhido);
                }
            },

            _onValueHelpConfirm: function (oEvent) {
                var oSelectedItem = oEvent.getParameter("selectedItem");
                if (!oSelectedItem) { return; }

                var sChaveEscolhida = oSelectedItem.getTitle();
                var sTextoEscolhido = oSelectedItem.getDescription();
                var oDialogModel = this.getView().getModel("modelDialog");

                if (this._sValueHelpTarget === "EMPRESA") {
                    oDialogModel.setProperty("/atribCNPJ", sChaveEscolhida);
                    oDialogModel.setProperty("/nomeEmpresaAtrib", sTextoEscolhido);
                } else if (this._sValueHelpTarget === "INSPETOR") {
                    oDialogModel.setProperty("/atribCPF", sChaveEscolhida);
                    oDialogModel.setProperty("/nomeInspetorAtrib", sTextoEscolhido);
                }
            },

            _onValueHelpCancel: function () {
                // Apenas fecha o pop-up sem ação extra
            },

            onCloseDialog: function () {
                this._bEstaSalvando = false;

                if (this._pEditDialog) {
                    this._pEditDialog.then(function (oDialog) {
                        if (oDialog && oDialog.isOpen()) {
                            oDialog.close();
                        }
                    });
                }
            },

            onValueHelpSearch: function (oEvent) {
                var sValue = oEvent.getParameter("value");
                var aFilters = [];

                var sCampoChave, sCampoTexto;

                if (this._sValueHelpTarget === "EMPRESA") {
                    sCampoChave = "Cnpj";
                    sCampoTexto = "Empresa";
                } else {
                    sCampoChave = "Cpf";
                    sCampoTexto = "Nome";
                }

                if (sValue) {
                    aFilters.push(
                        new Filter({
                            filters: [
                                new Filter(sCampoChave, FilterOperator.Contains, sValue),
                                new Filter(sCampoTexto, FilterOperator.Contains, sValue)
                            ],
                            and: false
                        })
                    );
                }

                var oBinding = oEvent.getSource().getBinding("items");
                oBinding.filter(aFilters);
            },


            onFormatCpfCnpj: function (oEvent) {
                var oInput = oEvent.getSource();
                var sValue = oEvent.getParameter("value") || "";
                var sPath = oInput.getBinding("value").getPath();


                var sFormatted = "";

                // Determina se o campo atual deve se comportar como CPF ou CNPJ
                var bIsCPF = false;

                if (sPath === "/atribCPF") {
                    bIsCPF = true;
                } else if (sPath === "/atribCNPJ") {
                    bIsCPF = false;
                } else if (sPath === "/chave") {
                    // Se o sPath for genérico (/chave), descobrimos pelo contexto do próprio Dialog.
                    // Exemplo: Se a label contiver "Cpf" ou se o comprimento máximo for atingido antes.
                    var oModel = this.getView().getModel("modelDialog");
                    var sLabel = oModel.getProperty("/labelChave") || "";

                    if (sLabel.toLowerCase().includes("cpf")) {
                        bIsCPF = true;
                    }
                }

                // Aplicação das regras de formatação baseadas na identificação
                if (bIsCPF) {

                    // Remove tudo o que não for dígito
                    var sNumbers = sValue.replace(/\D/g, "");

                    // Regra para CPF (Máximo 11 dígitos numéricos)
                    if (sNumbers.length > 11) {
                        sNumbers = sNumbers.substring(0, 11);
                    }

                    sFormatted = sNumbers
                        .replace(/(\d{3})(\d)/, "$1.$2")
                        .replace(/(\d{3})(\d)/, "$1.$2")
                        .replace(/(\d{3})(\d{1,2})$/, "$1-$2");
                } else {

                    var sAlfanumerico;
                     sAlfanumerico = sValue.replace(/[^a-zA-Z0-9]/g, "").toUpperCase();

                    // Regra para CNPJ (Máximo 14 dígitos numéricos)
                    if (sAlfanumerico.length > 14) {
                        sAlfanumerico = sAlfanumerico.substring(0, 14);
                    }

                    sFormatted = sAlfanumerico
                        .replace(/^(\w{2})(\w)/, "$1.$2")
                        .replace(/^(\w{2})\.(\w{3})(\w)/, "$1.$2.$3")
                        .replace(/\.(\w{3})(\w)/, ".$1/$2")
                        .replace(/(\w{4})(\w{1,2})$/, "$1-$2"); 
                }

                // Atualiza o valor do Input com a string devidamente mascarada
                oInput.setValue(sFormatted);
            }




        });
    });
