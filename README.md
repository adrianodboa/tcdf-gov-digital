import React, { useState, useMemo, useEffect } from "react";
import {
  Search,
  FileJson,
  Eye,
  Info,
  Tag,
  Calendar,
  FileText,
  Settings,
  Layers,
  X,
  Filter,
  Database,
  ArrowUpDown,
  ChevronLeft,
  ChevronRight,
  BarChart3,
  Download,
  Pencil,
  PlusCircle,
  LayoutGrid,
  CheckCircle2,
  Save,
  Loader2,
  AlertTriangle,
  Trash2,
} from "lucide-react";

// --- Dependências para o formulário ---
import { useForm, useFieldArray } from "react-hook-form";
import { z } from "zod";
import { zodResolver } from "@hookform/resolvers/zod";

// --- Dependências do Firebase ---
import { initializeApp } from "firebase/app";
import { getAuth, signInAnonymously, onAuthStateChanged } from "firebase/auth";
import {
  initializeFirestore,
  collection,
  onSnapshot,
  addDoc,
  doc,
  setDoc,
  deleteDoc,
} from "firebase/firestore";

// =====================================================================
// CHAVES DO FIREBASE CONFIGURADAS
// =====================================================================
const firebaseConfig = {
  apiKey: "AIzaSyAVsDBGduUOm4ztAIF_GL1RPOYJyPRvs0A",
  authDomain: "metadados-tcdf.firebaseapp.com",
  projectId: "metadados-tcdf",
  storageBucket: "metadados-tcdf.firebasestorage.app",
  messagingSenderId: "241154268712",
  appId: "1:241154268712:web:f0e9f8a2765f135be09f1d",
  measurementId: "G-5T1Q7NWQXL",
};

// Verificação de segurança: O código só tenta ligar à nuvem se tiver chaves reais
const isFirebaseConfigured = firebaseConfig.apiKey !== "COLE_AQUI";

let app, auth, db;
if (isFirebaseConfigured) {
  app = initializeApp(firebaseConfig);
  auth = getAuth(app);

  // MÁXIMA COMPATIBILIDADE COM FIREWALLS DE ÓRGÃOS PÚBLICOS:
  // Força requisições normais e desativa "streams" (que os firewalls costumam bloquear)
  db = initializeFirestore(app, {
    experimentalForceLongPolling: true,
    useFetchStreams: false,
  });
}

// --- DADOS INICIAIS FIXOS (Todos os 89 metadados) ---
const initialData = [
  {
    unidade: "DIFID",
    metadadoDeProcesso: true,
    tipoDocumento: ["Ofício", "Papel de Trabalho - PT"],
    nomeMetadado: "Número do Processo SEI",
    objetivoMetadado:
      "Identificação do número dos processos SEI mencionados nos documentos encaminhados pelo jurisdicionado. Costumamos resumir os docs SEI utilizados na análise em uma tabela no relatório e nos PTs. Resgatar essa informação exige abertura desses documentos.",
    formatoMetadado: "Número de Processo SEI",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: 1763151340720,
    id: 1,
    objetivoNaoInformado: false,
    imagens: [],
    precisaRevisao: false,
  },
  {
    unidade: "DIFID",
    metadadoDeProcesso: true,
    tipoDocumento: ["Ofício", "Papel de Trabalho - PT"],
    nomeMetadado: "Número do Processo Judicial",
    objetivoMetadado:
      "Identificação do número de processos judiciais mencionados nos documentos encaminhados pelo jurisdicionado que possam causar impacto no trabalho do TCDF. Essa informação é resgatada apenas com a abertura de relatórios ou PTs.",
    formatoMetadado: "Número de Processo Judicial",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: 1763151342783,
    id: 2,
    objetivoNaoInformado: false,
    imagens: [],
    precisaRevisao: false,
  },
  {
    unidade: "DIFID",
    metadadoDeProcesso: true,
    tipoDocumento: ["Ofício", "Papel de Trabalho - PT"],
    nomeMetadado: "CNPJ da Empresa Concessionária",
    objetivoMetadado:
      "Identificação das empresas concessionárias. Ex: empresas de ônibus (5 bacias) dificulta a identificação individualizada dos documentos.",
    formatoMetadado: "CNPJ",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: 1763151343320,
    id: 4,
    objetivoNaoInformado: false,
    imagens: [],
    precisaRevisao: false,
  },
  {
    unidade: "DIFID",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Recibo de Expediente",
      "Nota de Auditoria",
      "Nota de Inspeção",
      "Nota de Levantamento",
    ],
    nomeMetadado: "Órgão Destinatário",
    objetivoMetadado:
      "Identificação do órgão destinatário ou remetente. Atualmente é necessário abrir os documentos para identificação, o que complica quando temos múltiplos jurisdicionados em 1 mesmo processo. Ex: Levantamento em instrução na DIFID conta com dezenas de jurisdicionados (Proc. 4451/25).",
    formatoMetadado: "Lista controlada",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao:
      "Nesse caso a DIFID espera ter uma lista com todos os órgãos/entidades para o usuário selecionar. ",
    classeProcessual: [],
    timestamp: "2025-11-03T17:30:05.274Z",
    id: 5,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "DIFID",
    metadadoDeProcesso: false,
    tipoDocumento: ["Relatório/Voto", "Despacho Singular"],
    nomeMetadado: "Dias de Prorrogação de Prazo Concedidos",
    objetivoMetadado:
      "Necessidade de levantamento de dias de prorrogação concedidos para proposição de arquivamento de projetos muito antigos e defasados.",
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      "Para esse caso é necessário o metadado composto: CPF/CNPJ/Órgão + Quantidade de Dias Concedidos.",
    classeProcessual: [],
    timestamp: "2025-11-03T20:37:17.126Z",
    id: 6,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "DIFID",
    metadadoDeProcesso: true,
    tipoDocumento: ["Relatório"],
    nomeMetadado: "Tipo de Parceria",
    objetivoMetadado:
      "Identificação do tipo de parceria objeto da desestatização (PPP administrativa, PPP patrocinada, concessão, CDRU, ACT e outros tipos). Bastante útil para o processo, vez que todos os processos são classificados de forma genérica como PPP e Concessões. É necessário abrir o relatório para identificar o tipo específico de parceria a ser firmada.",
    formatoMetadado: "Lista controlada",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao:
      "Nesse caso é necessário uma lista (dropdown) com as opções padrões. O usuário selecionará apenas uma opção.",
    classeProcessual: [],
    timestamp: "2025-11-03T17:41:37.732Z",
    id: 7,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "DIFID",
    metadadoDeProcesso: true,
    tipoDocumento: ["Ofício", "Papel de Trabalho - PT"],
    nomeMetadado: "Número do Contrato",
    objetivoMetadado:
      "Identificação do número/ano dos contratos mencionados nos documentos encaminhados pelo jurisdicionado ou tratados nos PTs.",
    formatoMetadado: "Número de Contrato (XX/AAAA)",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-03T17:42:35.334Z",
    id: 8,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "DIFID",
    metadadoDeProcesso: true,
    tipoDocumento: ["Ofício", "Papel de Trabalho - PT"],
    nomeMetadado: "Número do Aditivo Contratual",
    objetivoMetadado:
      "Identificação do numero/ano dos aditivos contratuais mencionados nos documentos encaminhados pelo jurisdicionado ou tratados nos PTs.",
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      "Em contato com a Cinthia, ela informou que nesse caso ela precisa do número/ano do contrato + número/ano do termo aditivo para conseguir identificar a qual contrato pertence o aditivo. Sendo assim, a versão inicial (sem metadado composto) não atende.",
    classeProcessual: [],
    timestamp: "2025-11-03T20:37:40.134Z",
    id: 9,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "DIFID",
    metadadoDeProcesso: false,
    tipoDocumento: ["Relatório", "Informação", "Papel de Trabalho - PT"],
    nomeMetadado: "Rodada de Análise",
    objetivoMetadado:
      "Identificação de qual rodada de análise (ex:, 1, 2, 3, etc.) se refere aquele documento. Especialmente útil para PTs, que seguem numeração única por processo, sendo necessário controle paralelo para identificar qual sequencia de PT trata de qual rodada de análise.",
    formatoMetadado: "Número (inteiro)",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-03T18:54:55.690Z",
    id: 10,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "DIFID",
    metadadoDeProcesso: true,
    tipoDocumento: [],
    nomeMetadado: "Forma de Estruturação",
    objetivoMetadado:
      "Identificação da forma de estruturação do projeto (PMI, ACT, Cx, BNDES, MIP, Consultoria, etc.) em análise para avaliação estatística da taxa de sucesso dos projetos a depender da forma de estruturação adotada.",
    formatoMetadado: "Lista controlada",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao: "Necessário uma lista de seleção das opções pré-cadastradas.",
    classeProcessual: [],
    timestamp: "2025-11-03T20:27:50.822Z",
    id: 11,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Matriz de Responsabilização",
      "Informação",
      "Relatório FINAL DE AUDITORIA",
      "Decisão",
    ],
    nomeMetadado: "Responsável por Dano em Achado",
    objetivoMetadado: "Consolidação de dados da matriz de responsabilização",
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      "Metadado composto. Achado: Número; Responsável: CPF; Dano: R$.",
    classeProcessual: [],
    timestamp: "2025-11-03T20:37:04.190Z",
    id: 12,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Relatório FINAL DE AUDITORIA",
      "Relatório PRÉVIO DE AUDITORIA",
      "Informação",
      "Decisão",
    ],
    nomeMetadado: "Achado de Fiscalização",
    objetivoMetadado: "Permitir a pesquisa por tipo de achado.",
    formatoMetadado: "Texto livre",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação da SESPE: avaliar se fica melhor por tipo de irregularidade, cadastrar o achado ou já indicarmos aqui o tipo de irregularidade como metadado.",
    classeProcessual: [],
    timestamp: "2025-11-03T20:40:41.787Z",
    id: 13,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Relatório",
      "Relatório FINAL DE AUDITORIA",
      "Relatório PRÉVIO DE AUDITORIA",
      "Informação",
      "Despacho Singular",
      "Decisão",
      "Matriz de Achados",
      "Matriz de Responsabilização",
      "Matriz de Planejamento",
      "Relatório DE LPA",
    ],
    nomeMetadado: "Critério de Julgamento",
    objetivoMetadado:
      "Armazenar o critério de julgamento da licitação analisada.",
    formatoMetadado: "Texto livre",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-05T20:00:34.252Z",
    id: 14,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação", "Relatório DE LPA", "Decisão"],
    nomeMetadado: "Empresa",
    objetivoMetadado: "Montar uma base de dados de empresas fiscalizadas",
    formatoMetadado: "CNPJ",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-03T20:47:19.521Z",
    id: 15,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Relatório DE LPA",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório FINAL DE AUDITORIA",
      "Informação",
    ],
    nomeMetadado: "Existe Aditivo de Prazo?",
    objetivoMetadado:
      "Armazenar se há aditivo de prazo para o objeto fiscalizado por contrato.",
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      "Anotação Fabrício: Validado com Sílvia. Metadado composto. Necessário armazenar o número do contrato + indicador se existe aditivo de prazo.",
    classeProcessual: [],
    timestamp: 1762538699373,
    id: 16,
    objetivoNaoInformado: false,
    precisaRevisao: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Relatório DE LPA",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório FINAL DE AUDITORIA",
      "Informação",
    ],
    nomeMetadado: "Existe Aditivo de Quantidade?",
    objetivoMetadado:
      "Armazenar se há aditivo de quantidade para o objetivo fiscalizado, por contrato.",
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      "Anotação Fabrício: Validado com Sílvia. Metadado composto. Necessário armazenar o número do contrato e o indicador se há aditivo de quantidade.",
    classeProcessual: [],
    timestamp: 1762538698927,
    id: 17,
    objetivoNaoInformado: false,
    precisaRevisao: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "DETALHAMENTO DE BENEFÍCIOS",
      "Papel de Trabalho - PT",
      "DOCUMENTO DE AUDITORIA",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório FINAL DE AUDITORIA",
      "Memorial",
      "Matriz de Responsabilização",
      "Manifestação",
      "Decisão",
      "Matriz de Achados",
      "Despacho Singular",
      "Matriz de Planejamento",
      "Relatório DE LPA",
      "Relatório / VOTO",
      "Despacho",
    ],
    nomeMetadado: "Informação",
    objetivoMetadado:
      "Armazenar os e-docs das informações que estão relacionadas aos demais documentos.",
    formatoMetadado: "E-doc",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-05T20:12:02.230Z",
    id: 18,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Informação",
      "Relatório DE LPA",
      "Manifestação",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório FINAL DE AUDITORIA",
      "Nota de Auditoria",
      "RECIBO DE EXPEDIENTE",
      "RESPOSTA DE NOTA DE AUDITORIA",
      "Matriz de Responsabilização",
      "Matriz de Planejamento",
    ],
    nomeMetadado: "Jurisdicionado",
    objetivoMetadado:
      "Identificar o jurisdicionado que está sendo fiscalizado.",
    formatoMetadado: "CNPJ",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação da SESPE: Fragilidade do dado com relação a alterações na estrutura administrativa do DF. Recibo de expediente tramita em separado do processo de barramento. Já ocorreu por diversas vezes a perda do recibo ou seu envio para unidade errada pelo Protocolo.\nAnotação Fabrício: SEACOMP, DIFID e SEFIPE solicitaram uma lista (dropdown) com as opções de órgãos e entidades para seleção. Com a implantação da lista controlada, entendo que esse metadado da SESPE poderia fazer da mesma, facilitando o preenchimento e validação.",
    classeProcessual: [],
    timestamp: "2025-11-13T17:25:50.476Z",
    id: 19,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Relatório DE LPA",
      "Matriz de Planejamento",
      "Matriz de Responsabilização",
      "Relatório FINAL DE AUDITORIA",
      "Relatório PRÉVIO DE AUDITORIA",
      "Informação",
      "Matriz de Achados",
    ],
    nomeMetadado: "Modalidade de Licitação",
    objetivoMetadado:
      "Armazenar a modalidade de licitação utilizada na contratação (pregão, concorrência, convite, etc.).",
    formatoMetadado: "Lista controlada",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao:
      "Lista (dropdown) com as modalidades de licitação que poderão ser selecionadas.",
    classeProcessual: [],
    timestamp: "2025-11-05T20:20:57.145Z",
    id: 20,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "RESPOSTA DE NOTA DE AUDITORIA",
      "RECIBO DE EXPEDIENTE",
      "CÓPIA DE PROCESSO",
    ],
    nomeMetadado: "Nota de Auditoria",
    objetivoMetadado:
      "Armazenar a qual nota de auditoria se refere uma cópia de processo de barramento (por meio do qual foi enviada a nota de auditoria), um recibo de expediente assinado ou uma resposta de nota de auditoria. ",
    formatoMetadado: "E-doc",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação SESPE: Recibo de expediente tramita em separado do processo de barramento. Já ocorreu por diversas vezes a perda do recibo ou seu envio para unidade errada pelo Protocolo.",
    classeProcessual: [],
    timestamp: "2025-11-05T20:23:20.549Z",
    id: 21,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: true,
    tipoDocumento: [
      "Relatório DE LPA",
      "Matriz de Achados",
      "Informação",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório FINAL DE AUDITORIA",
      "Matriz de Responsabilização",
      "Matriz de Planejamento",
      "Manifestação",
      "Memorial",
    ],
    nomeMetadado: "Número do Contrato",
    objetivoMetadado:
      "Armazenar os número dos contratos que foram avaliados na fiscalização.",
    formatoMetadado: "Número de Contrato (XX/AAAA)",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Parcialmente",
    observacao:
      "Anotação Fabrício: o metadado a nível de documento será atendido com a versão inicial. Ficará pendente a parte de metadado de processo para as próximas versões.",
    classeProcessual: [],
    timestamp: "2025-11-05T20:33:18.906Z",
    id: 23,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: true,
    tipoDocumento: [
      "Relatório DE LPA",
      "Matriz de Achados",
      "Informação",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório FINAL DE AUDITORIA",
      "Matriz de Responsabilização",
      "Matriz de Planejamento",
      "Manifestação",
      "Memorial",
    ],
    nomeMetadado: "Número de Edital",
    objetivoMetadado:
      "Armazenar os números dos editais de licitação que foram fiscalizados.",
    formatoMetadado: "Número de Edital (XX/AAAA)",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Parcialmente",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-05T20:37:34.185Z",
    id: 24,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "RESPOSTA DE NOTA DE AUDITORIA",
      "RECIBO DE EXPEDIENTE",
      "Nota de Auditoria",
      "Ofício",
      "Manifestação",
      "CÓPIA DE PROCESSO",
    ],
    nomeMetadado: "Número do Processo de Barramento",
    objetivoMetadado:
      "Armazenar o número do processo de barramento que enviou uma nota de auditoria (que é copiado como peça para o e-TCDF atualmente) ou ofício de diligência saneadora ou que recebeu documentos externos (ex: manifestação, ofício, etc.).",
    formatoMetadado: "Número de Processo de Barramento",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação SESPE: Recibo de expediente tramita em separado do processo de barramento. Já ocorreu por diversas vezes a perda do recibo ou seu envio para unidade errada pelo Protocolo.\nAnotação Fabrício: O ofício (tipo de documento) é usado para envio de Ofício de Diligência Saneadora.",
    classeProcessual: [],
    timestamp: "2025-11-05T20:45:38.336Z",
    id: 25,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: true,
    tipoDocumento: [
      "Relatório DE LPA",
      "Matriz de Achados",
      "Informação",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório FINAL DE AUDITORIA",
      "Matriz de Responsabilização",
      "Matriz de Planejamento",
    ],
    nomeMetadado: "Número SIGGO do Contrato",
    objetivoMetadado:
      "Armazenar o número SIGGO dos contratos avaliados pela fiscalização. Esse número é diferente da numeração utilizada no documento de formalização do ajuste.",
    formatoMetadado: "Número (inteiro)",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Parcialmente",
    observacao:
      "Anotação da SESPE: avaliar se não está sobreposto ao número do contrato.\nAnotação Fabrício: feita validação com Marcelo (SESPE) que entende que não há sobreposição com o metadado de número de contrato.",
    classeProcessual: [],
    timestamp: "2025-11-05T20:36:07.126Z",
    id: 26,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Relatório DE LPA",
      "Matriz de Planejamento",
      "Matriz de Achados",
      "Matriz de Responsabilização",
      "Decisão",
      "Despacho Singular",
      "Informação",
      "Relatório FINAL DE AUDITORIA",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório",
      "Manifestação",
    ],
    nomeMetadado: "Objeto Fiscalizado",
    objetivoMetadado: "Armazenar o objeto fiscalizado.",
    formatoMetadado: "Texto livre",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "Anotação Fabrício: validado com Sílvia.",
    classeProcessual: [],
    timestamp: 1762538153075,
    id: 27,
    objetivoNaoInformado: false,
    precisaRevisao: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Relatório DE LPA",
      "Matriz de Planejamento",
      "Matriz de Responsabilização",
      "Relatório FINAL DE AUDITORIA",
      "Relatório PRÉVIO DE AUDITORIA",
      "Informação",
      "Matriz de Achados",
    ],
    nomeMetadado: "Prazo do Contrato (em dias)",
    objetivoMetadado:
      "Registrar o(s) prazo(s) do(s) contrato(s) fiscalizado(s) em dias.",
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Parcialmente",
    observacao:
      "Anotação da SESPE: pode ser composto no caso de processo relativo a mais de um contrato. \nAnotação Fabrício: até a implantação do metadado composto, será mantido como metadado simples (atende a maioria dos casos da SESPE). Após a implantação, será feita a modificação e migração dos dados.",
    classeProcessual: [],
    timestamp: "2025-11-05T20:53:04.154Z",
    id: 28,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Decisão",
      "Despacho Singular",
      "Nota de Auditoria",
      "Ofício",
    ],
    nomeMetadado: "Prazo para Cumprimento",
    objetivoMetadado:
      "Armazenar o prazo estabelecido para obter a resposta do jurisdicionado ou para cumprimento do que foi determinado.",
    formatoMetadado: "Data/Hora",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-05T20:54:52.610Z",
    id: 29,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: ["Nota de Auditoria", "Ofício"],
    nomeMetadado: "RECIBO DE EXPEDIENTE",
    objetivoMetadado:
      "Armazenar o e-doc do recibo de expediente que entregou o Ofício (de diligência saneadora) ou a Nota de Auditoria à parte/órgão.",
    formatoMetadado: "E-doc",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-05T20:55:25.760Z",
    id: 30,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "TERMO DE AUTUAÇÃO",
      "Informação",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório FINAL DE AUDITORIA",
      "Relatório DE LPA",
    ],
    nomeMetadado: "Referências ao PGA e PGF",
    objetivoMetadado:
      "Armazenar a qual PGF e PGA (processos do TCDF) se refere determinado processo ou fiscalização.",
    formatoMetadado: "Número de Processo TCDF",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-05T20:56:00.181Z",
    id: 31,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Relatório DE LPA",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório FINAL DE AUDITORIA",
      "Informação",
      "Matriz de Achados",
      "Matriz de Responsabilização",
      "Matriz de Planejamento",
    ],
    nomeMetadado: "Regime de Execução",
    objetivoMetadado:
      "Armazenar o regime de execução dos contratos avaliados de acordo com os regimes previstos na Lei de Licitações (empreitada por preço unitário, empreitada por preço global, empreitada integral, tarefa, contratação integrada, contratação semi-integrada, fornecimento e prestação de serviço associado).",
    formatoMetadado: "Lista controlada",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Parcialmente",
    observacao:
      "Anotação Fabrício: até a implantação do metadado composto, será mantido como metadado simples (atende a maioria dos casos da SESPE). Após a implantação, será feita a modificação e migração dos dados.",
    classeProcessual: [],
    timestamp: "2025-11-05T20:59:06.028Z",
    id: 32,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Papel de Trabalho - PT",
      "DOCUMENTO DE AUDITORIA",
      "Matriz de Achados",
      "Matriz de Planejamento",
      "Matriz de Responsabilização",
      "Manifestação",
      "Informação",
      "Decisão",
      "Despacho Singular",
      "Memorial",
      "DETALHAMENTO DE BENEFÍCIOS",
    ],
    nomeMetadado: "Relatório de Auditoria",
    objetivoMetadado:
      "Armazenar o e-doc do Relatório de Auditoria (prévio ou final, a depender da fase do processo) que está relacionado aos demais documentos.",
    formatoMetadado: "E-doc",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-05T21:00:32.193Z",
    id: 33,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Matriz de Achados",
      "Matriz de Responsabilização",
      "Matriz de Planejamento",
      "Decisão",
      "Despacho Singular",
      "Informação",
      "Relatório FINAL DE AUDITORIA",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório",
    ],
    nomeMetadado: "Tipo de Irregularidade",
    objetivoMetadado:
      "Armazenar o tipo de irregularidade por temas (habilitação técnica, qualidade da obra, reajuste, etc.).",
    formatoMetadado: "Lista controlada",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao: "Anotação SESPE: validado com Silvia.",
    classeProcessual: [],
    timestamp: 1762538218424,
    id: 34,
    objetivoNaoInformado: false,
    precisaRevisao: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Relatório FINAL DE AUDITORIA",
      "Relatório PRÉVIO DE AUDITORIA",
      "Informação",
      "Relatório DE LPA",
      "Despacho Singular",
      "Decisão",
      "Matriz de Responsabilização",
      "Matriz de Planejamento",
      "Matriz de Achados",
    ],
    nomeMetadado: "Tipo de Obra ou Serviço",
    objetivoMetadado:
      "Armazenar a tipologia do objeto (obra de drenagem, pavimentação, edificação, saneamento, etc.).",
    formatoMetadado: "Lista controlada",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao: "Anotação Fabrício: validado com Sílvia.",
    classeProcessual: [],
    timestamp: 1762538271036,
    id: 35,
    objetivoNaoInformado: false,
    precisaRevisao: false,
    imagens: [],
  },
  {
    unidade: "SESPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Relatório DE LPA",
      "DETALHAMENTO DE BENEFÍCIOS",
      "Informação",
      "Matriz de Responsabilização",
      "Matriz de Planejamento",
      "Matriz de Achados",
      "Relatório FINAL DE AUDITORIA",
      "Relatório PRÉVIO DE AUDITORIA",
    ],
    nomeMetadado: "Valor do Contrato",
    objetivoMetadado:
      "Registrar o(s) valor(es) de formalização do(s) contrato(s) fiscalizados.",
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Parcialmente",
    observacao:
      "Anotação SESPE: pode ser composto no caso de processo relativo a mais de um contrato.\nAnotação Fabrício: até a implantação do metadado composto, será mantido como metadado simples (atende a maioria dos casos da SESPE). Após a implantação, será feita a modificação e migração dos dados.",
    classeProcessual: [],
    timestamp: "2025-11-05T20:54:07.233Z",
    id: 36,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEACOMP",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação", "NOTA DE INSPEÇÃO", "RELATÓRIO DE INSPEÇÃO"],
    nomeMetadado: "Jurisdicionado",
    objetivoMetadado: "Indicar o órgão ou entidade analisado.",
    formatoMetadado: "Lista controlada",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao:
      "Anotação Fabrício: a área espera uma lista de seleção com os órgãos/entidades. Uma solução inicial seria permitir o campo em texto livre mas isso causaria problemas de migração dos dados depois de implantada nova versão que suporte listas.",
    classeProcessual: [],
    timestamp: "2025-11-03T21:50:39.974Z",
    id: 37,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEACOMP",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação", "NOTA DE INSPEÇÃO", "RELATÓRIO DE INSPEÇÃO"],
    nomeMetadado: "Tema da Fiscalização",
    objetivoMetadado:
      "Armazenar e classificar a matéria sob exame em macrotemas padronizados.",
    formatoMetadado: "Lista controlada",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao:
      'Anotação da SEACOMP: Lista contendo principais temas tratados: "Licitações"; "Contratos"; "Parcerias - MROSC”; “Parcerias - Outras"; “Obras e Serviços de Engenharia” "Gestão Pública”; “Política Pública” “Transparência” “Outros”.',
    classeProcessual: [],
    timestamp: "2025-11-03T21:56:39.390Z",
    id: 38,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEACOMP",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação", "NOTA DE INSPEÇÃO", "RELATÓRIO DE INSPEÇÃO"],
    nomeMetadado: "Instrumento Fiscalizado",
    objetivoMetadado:
      "Registrar, de forma padronizada, o instrumento sob fiscalização — tipo (Edital/Dispensa/Contrato/ARP/Termo etc.), número/ano e órgão/unidade emitente.",
    formatoMetadado: "Texto livre",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      'Anotação da SEACOMP: exemplo "Contrato nº 047290/2024 - SES/DF"; "Dispensa de Licitação nº 50/2025 – SEE/DF".\nAnotação Fabrício: outras áreas demandaram os metadados Número do Contrato para identificar os contratos fiscalizados (armazenando somente XX/AAAA) e Número do Edital (para os editais), a exemplo da SESPE.',
    classeProcessual: [],
    timestamp: "2025-11-05T20:39:22.391Z",
    id: 39,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEACOMP",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação", "NOTA DE INSPEÇÃO", "RELATÓRIO DE INSPEÇÃO"],
    nomeMetadado: "Objeto Fiscalizado",
    objetivoMetadado:
      "Registrar, de forma padronizada, a descrição do objeto sob exame (licitação, contrato, obra/serviço ou fato representado/denunciado).",
    formatoMetadado: "Texto livre",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      'Anotação da SEACOMP: exemplo "Compra de medicamentos"; "Serviços de manutenção predial"; "Fila de procedimentos cirúrgicos"; "Falta de médicos"; “Transporte Público Coletivo” “Fornecimento de Merenda Escolar”.',
    classeProcessual: [],
    timestamp: "2025-11-03T21:56:13.958Z",
    id: 40,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEACOMP",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação", "NOTA DE INSPEÇÃO", "RELATÓRIO DE INSPEÇÃO"],
    nomeMetadado: "Data da Irregularidade",
    objetivoMetadado:
      "Registrar a data da ocorrência da suposta irregularidade (em caso de ato instantâneo) ou da data em que a infração cessou (em caso de infração permanente ou continuada), auxiliando na contagem de prazos/determinações e seu status.",
    formatoMetadado: "Data",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: tenho dúvida em relação à multiplicidade desse metadado. No caso de mais de uma irregularidade ser mencionada nos documentos, cada uma com uma data de ocorrência, como ficaria? ",
    classeProcessual: [],
    timestamp: "2025-11-03T21:58:56.572Z",
    id: 41,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEACOMP",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação", "NOTA DE INSPEÇÃO", "RELATÓRIO DE INSPEÇÃO"],
    nomeMetadado: "Empresa Envolvida",
    objetivoMetadado:
      "Identificar as pessoas jurídicas relacionadas ao fato sob exame (licitante, contratada, OSC, consorciada etc.).",
    formatoMetadado: "CNPJ",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: outras áreas pediram metadado similar com o mesmo nome (Empresa, Empresa Concessionária, etc.).",
    classeProcessual: [],
    timestamp: "2025-11-03T22:01:37.473Z",
    id: 42,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEACOMP",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação", "NOTA DE INSPEÇÃO", "RELATÓRIO DE INSPEÇÃO"],
    nomeMetadado: "Pessoa Física Envolvida",
    objetivoMetadado:
      "Identificar as pessoas físicas relacionadas ao fato sob exame (responsável, gestor, signatário, relator etc.)",
    formatoMetadado: "CPF",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-03T22:02:33.162Z",
    id: 43,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEAUD",
    metadadoDeProcesso: false,
    tipoDocumento: ["Relatório / VOTO"],
    nomeMetadado: "Orientação do Voto Condutor",
    objetivoMetadado:
      "Registrar as situações em que o Plenário segue ou não as  sugestões do Corpo Técnico, para aprimoramento e elaboração de indicadores.",
    formatoMetadado: "Lista controlada",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao:
      "Anotação SEAUD: 1. Convergente com o Corpo Técnico; 2. Parcialmente convergente com o Corpo Técnico; 3. Divergente.",
    classeProcessual: [],
    timestamp: "2025-11-03T22:05:27.409Z",
    id: 44,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEAUD",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Informação",
      "Relatório",
      "RELATÓRIO DE INSPEÇÃO",
      "Matriz de Achados",
      "Matriz de Responsabilização",
    ],
    nomeMetadado: "Irregularidade",
    objetivoMetadado:
      "Registrar as irregularidades detectadas em instruções feitas fora do Sisaudit, a exemplo das registradas em matriz de achado e/ou relatórios produzidos pela Controladoria-Geral do Distrito Federal.",
    formatoMetadado: "Texto livre",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: A SEAUD avalia relatórios encaminhados pela CGDF, razão pela qual precisa de metadados para esses documentos, os quais, muitas vezes, chegam cadastrados no e-TCDF com o tipo de documento errado. ",
    classeProcessual: [],
    timestamp: "2025-11-03T22:11:30.015Z",
    id: 45,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEAUD",
    metadadoDeProcesso: false,
    tipoDocumento: ["Relatório", "Matriz de Responsabilização"],
    nomeMetadado: "Responsável (CPF)",
    objetivoMetadado:
      "Armazenar as pessoas físicas indicadas como responsáveis em documentos da Controladoria-Geral do Distrito Federal.",
    formatoMetadado: "CPF",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: Há metadado similar pedido por outras áreas. A SEAUD avalia relatórios encaminhados pela CGDF, razão pela qual precisa de metadados para esses documentos, os quais, muitas vezes, chegam cadastrados no e-TCDF com o tipo de documento errado. ",
    classeProcessual: [],
    timestamp: "2025-11-03T22:10:09.981Z",
    id: 46,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEAUD",
    metadadoDeProcesso: false,
    tipoDocumento: ["Relatório", "Matriz de Responsabilização"],
    nomeMetadado: "Responsável (CNPJ)",
    objetivoMetadado:
      "Armazenar as pessoas jurídicas indicadas como responsáveis em documentos da Controladoria-Geral do Distrito Federal.",
    formatoMetadado: "CNPJ",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: Há metadado similar pedido por outras áreas. A SEAUD avalia relatórios encaminhados pela CGDF, razão pela qual precisa de metadados para esses documentos, os quais, muitas vezes, chegam cadastrados no e-TCDF com o tipo de documento errado. ",
    classeProcessual: [],
    timestamp: "2025-11-05T16:52:11.384Z",
    id: 47,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEAUD",
    metadadoDeProcesso: false,
    tipoDocumento: ["Relatório"],
    nomeMetadado: "Temática da Auditoria",
    objetivoMetadado:
      "Identificar a temática da auditoria feita pela CGDF, usando a mesma lógica da temática de processes do TCDF (tesauro, com ajustes).",
    formatoMetadado: "Texto livre",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: o tipo de documento Relatório é utilizado hoje para armazenar relatórios de auditoria produzidos pela CGDF e encaminhados ao TCDF. A SEAUD recebe esses documentos para avaliação.",
    classeProcessual: [],
    timestamp: "2025-11-03T22:12:53.541Z",
    id: 48,
    objetivoNaoInformado: false,
    imagens: [],
  },
  {
    unidade: "SEAUD",
    metadadoDeProcesso: true,
    tipoDocumento: [],
    nomeMetadado: "Número do Contrato",
    objetivoMetadado: "Registrar os contratos analisados no processo. ",
    objetivoNaoInformado: false,
    formatoMetadado: "Número de Contrato (XX/AAAA)",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao:
      "Anotação SEAUD: importante que esse campo não seja do tipo texto, mas uma lista a partir dos números de contratos registrados no SIGGO.\nAnotação Fabrício: essa validação no SIGGO seria com o número SIGGO do contrato ou número do contrato em si? Poderia ser feita assim que o usuário terminar de digitar para evitar ter que montar uma lista controlada (dropdown) imenso.",
    classeProcessual: [],
    timestamp: "2025-11-05T16:56:21.020Z",
    id: 49,
    imagens: [],
  },
  {
    unidade: "SEAUD",
    metadadoDeProcesso: true,
    tipoDocumento: [],
    nomeMetadado: "Número do Processo SEI",
    objetivoMetadado:
      "Registrar os números dos processos SEI analisado no processo do e-TCDF",
    objetivoNaoInformado: false,
    formatoMetadado: "Número de Processo SEI",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao:
      "Anotação SEAUD: também seria importante validar o número do processo caso se tenha acesso de API ou outra forma de validação à base de dados do SEI. Destaca-se que esse recurso existia anteriormente no e-TCDF, quando os processos eram físicos, funcionalidade que se perdeu ao migrar para processos eletrônicos.",
    classeProcessual: [],
    timestamp: "2025-11-05T17:00:57.046Z",
    id: 50,
    imagens: [],
  },
  {
    unidade: "SEAUD",
    metadadoDeProcesso: true,
    tipoDocumento: [],
    nomeMetadado: "Função de Governo",
    objetivoMetadado:
      "Registrar a função de governo fiscalizada para permitir agregar informação de forma mais rápida.",
    objetivoNaoInformado: false,
    formatoMetadado: "Lista controlada",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao:
      "Anotação Fabrício: atualmente são 27 funções de governo. O usuário selecionará a opção de uma lista fechada.",
    classeProcessual: [],
    timestamp: "2025-11-05T17:09:40.626Z",
    id: 51,
    imagens: [],
  },
  {
    unidade: "SEAUD",
    metadadoDeProcesso: true,
    tipoDocumento: [],
    nomeMetadado: "Temática do Processo",
    objetivoMetadado: "Registrar a temática avaliada no processo.",
    objetivoNaoInformado: false,
    formatoMetadado: "Lista controlada",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao:
      "Anotação SEAUD: acreditamos que o campo Tesauro pode ser usado para essa finalidade, mas sua lógia de preenchimento precisa ser reformulada (talvez seja necessário criar uma lista mais fechada de temáticas). Uma boa referência ou ponto de partida é a classificação de processo - raciocínio semelhante pode ser feito, partindo da função de governo e detalhando as temáticas de fiscalização envolvidas.",
    classeProcessual: [],
    timestamp: "2025-11-05T17:17:54.169Z",
    id: 52,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: ["RECIBO DE DOCUMENTO", "RECIBO DE EXPEDIENTE"],
    nomeMetadado: "Data/Hora do Recebimento",
    objetivoMetadado: "Armazenar a data de hora do recebimento da comunicação.",
    objetivoNaoInformado: false,
    formatoMetadado: "Data/Hora",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Verificar se e-TCDF já possui esse campo para os tipos de documento mencionados.",
    classeProcessual: [],
    timestamp: "2025-11-10T19:22:57.639Z",
    id: 53,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "DOCUMENTO PARTICULAR",
      "PETIÇÃO",
      "REQUERIMENTO",
      "SOLICITAÇÃO",
      "Manifestação",
      "OUTROS",
      "EMBARGOS",
      "EMBARGOS DE DECLARAÇÃO",
      "DEFESA",
      "DEFESA PRÉVIA",
      "ALEGAÇÕES",
      "RAZÕES DE JUSTIFICATIVA",
      "CONTRARRAZÕES",
      "ESCLARECIMENTO",
      "PEDIDO DE RECONSIDERAÇÃO",
      "PEDIDO DE SUSTENTAÇÃO ORAL",
      "RECURSO",
      "REPRESENTAÇÃO CONJUNTA",
      "REPRESENTAÇÃO",
    ],
    nomeMetadado:
      "Contém procuração, substabelecimento ou renúncia de poderes?",
    objetivoMetadado:
      'Indicar documentos que não sejam do tipo "Procuração" mas que contenham procuração, substabelecimento com/sem reserva de poderes ou renúncia de poderes em seu conteúdo.',
    objetivoNaoInformado: false,
    formatoMetadado: "Booleano (SIM/NÃO)",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Pode ser utilizado na funcionalidade de cadastro de partes e representantes para limitar a lista de documentos exibida para os usuários na hora do cadastramento de uma nova procuração, substabelecimento ou renúncia de poderes.",
    classeProcessual: [],
    timestamp: "2025-11-10T19:27:22.864Z",
    id: 54,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "DOCUMENTO PARTICULAR",
      "PETIÇÃO",
      "REQUERIMENTO",
      "SOLICITAÇÃO",
      "Manifestação",
      "OUTROS",
      "CONTRARRAZÕES",
      "ESCLARECIMENTO",
      "PEDIDO DE RECONSIDERAÇÃO",
      "RECURSO",
      "RAZÕES DE JUSTIFICATIVA",
      "REPRESENTAÇÃO",
      "REPRESENTAÇÃO CONJUNTA",
      "DEFESA",
      "DEFESA PRÉVIA",
      "ALEGAÇÕES",
    ],
    nomeMetadado: "Contém pedido de sustentação oral?",
    objetivoMetadado:
      'Indicar para o sistema documentos que contenham pedido de sustentação oral em seu conteúdo e que não sejam do tipo "PEDIDO DE SUSTENTAÇÃO ORAL".',
    objetivoNaoInformado: false,
    formatoMetadado: "Booleano (SIM/NÃO)",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-10T19:29:04.842Z",
    id: 55,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "DOCUMENTO PARTICULAR",
      "PETIÇÃO",
      "REQUERIMENTO",
      "SOLICITAÇÃO",
      "Manifestação",
      "OUTROS",
    ],
    nomeMetadado: "Contém pedido de prorrogação de prazo?",
    objetivoMetadado:
      "Indicar documentos que contenham pedido de prorrogação de prazo em seu conteúdo.",
    objetivoNaoInformado: false,
    formatoMetadado: "Booleano (SIM/NÃO)",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-10T19:30:28.081Z",
    id: 56,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "DOCUMENTO PARTICULAR",
      "PETIÇÃO",
      "REQUERIMENTO",
      "SOLICITAÇÃO",
      "Manifestação",
      "OUTROS",
    ],
    nomeMetadado: "Contém pedido de cópia/vista dos autos?",
    objetivoMetadado:
      "Indicar para o sistema documentos que contenham pedido de cópia ou vistas em seu conteúdo.",
    objetivoNaoInformado: false,
    formatoMetadado: "Booleano (SIM/NÃO)",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-10T19:31:26.287Z",
    id: 57,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: ["ACÓRDÃO"],
    nomeMetadado: "Tipo de Acórdão",
    objetivoMetadado:
      "Indicar o tipo de acórdão lavrado (regularidade com quitação plena, regularidade com ressalva, irregularidade com imputação de débito, irregularidade sem débito mas com multa, contas iliquidáveis/encerramento, aplicação de multa, tornar sem efeito acórdão, quitação de débito/multa).",
    objetivoNaoInformado: false,
    formatoMetadado: "Lista controlada",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-10T19:45:44.817Z",
    id: 58,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: ["OFÍCIO GP"],
    nomeMetadado: "E-doc do Despacho Singular Comunicado",
    objetivoMetadado:
      "Identificar qual despacho singular está sendo comunicado pelo Ofício GP (quando for o caso).",
    objetivoNaoInformado: false,
    formatoMetadado: "E-doc",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-10T19:49:31.738Z",
    id: 59,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: ["PUBLICAÇÃO DO(S) ACORDÃO(S)"],
    nomeMetadado: "E-doc do Acórdão",
    objetivoMetadado:
      'Armazenar quais acórdãos foram publicados para o tipo de documento "PUBLICAÇÃO DO(S) ACÓRDÃO(S)".',
    objetivoNaoInformado: false,
    formatoMetadado: "E-doc",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Alternativamente poderia ser utilizado o número/ano do acórdão.",
    classeProcessual: [],
    timestamp: "2025-11-10T19:52:26.040Z",
    id: 60,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "OFÍCIO GP",
      "CIENTIFICAÇÃO",
      "NOTIFICAÇÃO",
      "CITAÇÃO",
      "COMUNICAÇÃO DE AUDIÊNCIA",
    ],
    nomeMetadado: "E-doc da Decisão Comunicada",
    objetivoMetadado: "Armazenar o e-doc da decisão que está sendo comunicada.",
    objetivoNaoInformado: false,
    formatoMetadado: "E-doc",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Alternativamente poderia ser utilizado o número/ano da decisão.",
    classeProcessual: [],
    timestamp: "2025-11-10T19:53:43.335Z",
    id: 61,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: ["RECIBO DE DOCUMENTO", "RECIBO DE EXPEDIENTE"],
    nomeMetadado: "E-doc da Comunicação Original",
    objetivoMetadado:
      "Armazenar o e-doc da comunicação original (ex: do Ofício GP) para os recibos de documento ou de expediente.",
    objetivoNaoInformado: false,
    formatoMetadado: "E-doc",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "Verificar se e-TCDF já realiza esse vínculo automaticamente.",
    classeProcessual: [],
    timestamp: "2025-11-10T19:54:53.037Z",
    id: 62,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: ["PUBLICAÇÃO DO(S) ACORDÃO(S)"],
    nomeMetadado: "Data da Publicação no Diário Oficial",
    objetivoMetadado:
      'Armazenar a data de publicação do(s) acórdãos no Diário Oficial para o tipo de documento "PUBLICAÇÃO DO(S) ACÓRDÃO(S)”.',
    objetivoNaoInformado: false,
    formatoMetadado: "Data",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-10T19:55:58.335Z",
    id: 63,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "ACÓRDÃO",
      "Matriz de Responsabilização",
      "DEMONSTRATIVO DE ATUALIZAÇÃO DE VALORES - SINDEC",
    ],
    nomeMetadado: "Responsável (CNPJ)",
    objetivoMetadado:
      "Armazenar as pessoas jurídicas indicadas como responsáveis.",
    objetivoNaoInformado: false,
    formatoMetadado: "CNPJ",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-10T20:11:27.295Z",
    id: 64,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "ACÓRDÃO",
      "Matriz de Responsabilização",
      "DEMONSTRATIVO DE ATUALIZAÇÃO DE VALORES - SINDEC",
    ],
    nomeMetadado: "Responsável (CPF)",
    objetivoMetadado:
      "Armazenar as pessoas físicas indicadas como responsáveis.",
    objetivoNaoInformado: false,
    formatoMetadado: "CPF",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-10T20:11:41.499Z",
    id: 65,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: ["OFÍCIO GP", "CITAÇÃO", "CIENTIFICAÇÃO"],
    nomeMetadado: "Destinatário (CNPJ)",
    objetivoMetadado:
      "Armazenar o destinatário de uma comunicação do TCDF, quando se tratar de pessoa jurídica.",
    objetivoNaoInformado: false,
    formatoMetadado: "CNPJ",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Nesse caso, além do destinatário (CNPJ) também deverá ser informado o destinatário (CPF) que será o representante legal autorizado a receber a comunicação em nome da empresa.",
    classeProcessual: [],
    timestamp: "2025-11-10T20:00:04.953Z",
    id: 66,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "OFÍCIO GP",
      "CITAÇÃO",
      "CIENTIFICAÇÃO",
      "NOTIFICAÇÃO",
      "COMUNICAÇÃO DE AUDIÊNCIA",
    ],
    nomeMetadado: "Destinatário (CPF)",
    objetivoMetadado:
      "Armazenar o destinatário de uma comunicação do TCDF, quando se tratar de pessoa física.",
    objetivoNaoInformado: false,
    formatoMetadado: "CPF",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    timestamp: "2025-11-10T20:00:59.014Z",
    id: 67,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: ["RECIBO DE DOCUMENTO", "RECIBO DE EXPEDIENTE"],
    nomeMetadado: "Responsável pelo Recebimento (CPF)",
    objetivoMetadado:
      "Armazenar o CPF do responsável pelo recebimento de uma comunicação do TCDF, inclusive no caso de advogados.",
    objetivoNaoInformado: false,
    formatoMetadado: "CPF",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "Verificar se o e-TCDF já armazena essa informação. ",
    classeProcessual: [],
    timestamp: "2025-11-10T20:02:31.708Z",
    id: 68,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "DOCUMENTO PARTICULAR",
      "PETIÇÃO",
      "REQUERIMENTO",
      "SOLICITAÇÃO",
      "Manifestação",
      "OUTROS",
      "EMBARGOS",
      "EMBARGOS DE DECLARAÇÃO",
      "REQUISIÇÃO",
      "DEFESA",
      "DEFESA PRÉVIA",
      "ALEGAÇÕES",
      "RAZÕES DE JUSTIFICATIVA",
      "CONTRARRAZÕES",
      "ESCLARECIMENTO",
      "PEDIDO DE RECONSIDERAÇÃO",
      "PEDIDO DE SUSTENTAÇÃO ORAL",
      "REPRESENTAÇÃO",
      "REPRESENTAÇÃO CONJUNTA",
      "CONSULTA",
    ],
    nomeMetadado: "Autor (CPF)",
    objetivoMetadado:
      "Armazenar as pessoas físicas responsáveis pela autoria de um documento submetido ao TCDF (ex: petição).",
    objetivoNaoInformado: false,
    formatoMetadado: "CPF",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: entendo que esse metadado é de caráter geral e poderia ser utilizado pelas demais áreas.",
    classeProcessual: [],
    timestamp: "2025-11-10T20:06:39.521Z",
    id: 69,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "DOCUMENTO PARTICULAR",
      "PETIÇÃO",
      "REQUERIMENTO",
      "SOLICITAÇÃO",
      "Manifestação",
      "OUTROS",
      "EMBARGOS",
      "EMBARGOS DE DECLARAÇÃO",
      "REQUISIÇÃO",
      "DEFESA",
      "DEFESA PRÉVIA",
      "ALEGAÇÕES",
      "CONTRARRAZÕES",
      "ESCLARECIMENTO",
      "PEDIDO DE RECONSIDERAÇÃO",
      "PEDIDO DE SUSTENTAÇÃO ORAL",
      "REPRESENTAÇÃO",
      "REPRESENTAÇÃO CONJUNTA",
      "CONSULTA",
    ],
    nomeMetadado: "Autor (CNPJ)",
    objetivoMetadado:
      "Armazenar as pessoas jurídicas responsáveis pela autoria de um documento submetido ao TCDF (ex: petição).",
    objetivoNaoInformado: false,
    formatoMetadado: "CNPJ",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: entendo que esse metadado é de caráter geral e poderia ser utilizado pelas demais áreas.",
    classeProcessual: [],
    timestamp: "2025-11-10T20:07:54.328Z",
    id: 70,
    imagens: [],
  },
  {
    unidade: "SECONT",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "DOCUMENTO PARTICULAR",
      "PETIÇÃO",
      "REQUERIMENTO",
      "SOLICITAÇÃO",
      "Manifestação",
      "OUTROS",
      "EMBARGOS",
      "EMBARGOS DE DECLARAÇÃO",
      "DEFESA",
      "DEFESA PRÉVIA",
      "ALEGAÇÕES",
      "RAZÕES DE JUSTIFICATIVA",
      "CONTRARRAZÕES",
      "ESCLARECIMENTO",
      "PEDIDO DE RECONSIDERAÇÃO",
      "PEDIDO DE SUSTENTAÇÃO ORAL",
      "RECURSO",
      "REPRESENTAÇÃO",
      "REPRESENTAÇÃO CONJUNTA",
      "Memorial",
    ],
    nomeMetadado: "Signatário (CPF)",
    objetivoMetadado:
      "Armazenar os signatários de um documento encaminhado ao TCDF, inclusive no caso de advogados.",
    objetivoNaoInformado: false,
    formatoMetadado: "CPF",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: entendo que esse metadado é de caráter geral e poderia ser utilizado pelas demais áreas.",
    classeProcessual: [],
    timestamp: "2025-11-10T20:10:31.557Z",
    id: 71,
    imagens: [],
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: ["EXTRATO SIRAC"],
    nomeMetadado: "Ato de Concessão",
    objetivoMetadado:
      "Armazenar metadados de cada ato de concessão constante do extrato SIRAC.",
    objetivoNaoInformado: false,
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      'Estrutura dos metadados compostos para cada ato de concessão (conforme imagem):\n\na. Número do ato: Código de identificação único do ato administrativo. (Tipo: número formatado)\nb. Indicação: Classificação do ato quanto à sua situação, como "Legalidade", “Ilegalidade”, etc. (Tipo: Lista controlada)\nc. Data de vigência: Data a partir da qual o ato passa a ter validade legal. (Tipo: Data)\nd. Data de publicação: Data em que o ato foi publicado em um diário ou meio oficial. (Tipo: Data)\ne. Montante em Exame: Valor monetário do benefício que está sob análise. (Tipo: valor R$)\nf. Tipo de Ato: Natureza específica do ato, como "Pensão Civil". (Tipo: Lista controlada)\n\ng. Servidor/Instituidor: metadados da pessoa que originou o direito ao benefício. É obrigatório e admite uma ou mais ocorrências (1+).\ng.1. CPF: Cadastro de Pessoa Física do servidor. (Tipo: CPF)\ng.2. Matrícula: Número da matrícula funcional do servidor no órgão. (Tipo: número inteiro)\ng.3. Órgão: Sigla ou nome do órgão ao qual o servidor era vinculado. (Tipo: texto livre)\ng.4. Nome: Nome completo do servidor. (Tipo: texto livre)\ng.5. Cargo: Cargo que o servidor ocupava. (Tipo: texto livre)\n\nh. Pensionista: identifica cada beneficiário da pensão. É opcional e admite múltiplas ocorrências (0+). Se o grupo "Pensionista" existir, seus metadados são obrigatórios.\nh.1. CPF: Cadastro de Pessoa Física do pensionista. (Tipo: CPF)\nh.2. Matrícula: Número da matrícula do pensionista (se houver). (Tipo: número inteiro)\nh.3. Órgão: Órgão de vínculo do pensionista. (Tipo: texto livre)\nh.4. Nome: Nome completo do pensionista. (Tipo: texto livre)\nh.5. Vínculo: Relação de parentesco do pensionista com o servidor instituidor (ex: Cônjuge, Filho). (Tipo: Lista controlada)',
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-12T18:18:19.837Z",
    id: 72,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "OFÍCIO GP",
      "CIENTIFICAÇÃO",
      "NOTIFICAÇÃO",
      "CITAÇÃO",
      "COMUNICAÇÃO DE AUDIÊNCIA",
    ],
    nomeMetadado: "E-doc da Decisão Comunicada",
    objetivoMetadado: "Armazenar o e-doc da decisão que está sendo comunicada.",
    objetivoNaoInformado: false,
    formatoMetadado: "E-doc",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Alternativamente poderia ser utilizado o número/ano da decisão.",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:01:45.111Z",
    id: 73,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "OFÍCIO GP",
      "CITAÇÃO",
      "CIENTIFICAÇÃO",
      "NOTIFICAÇÃO",
      "COMUNICAÇÃO DE AUDIÊNCIA",
    ],
    nomeMetadado: "Destinatário (CPF)",
    objetivoMetadado:
      "Armazenar o destinatário de uma comunicação do TCDF, quando se tratar de pessoa física.",
    objetivoNaoInformado: false,
    formatoMetadado: "CPF",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:01:35.431Z",
    id: 74,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação"],
    nomeMetadado: "Servidor",
    objetivoMetadado:
      "Permitir recuperar documentos relativos a um servidor, empregado público ou militar.",
    objetivoNaoInformado: false,
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      "Metadado composto: CPF, Matrícula do Servidor (00000-DV ou Texto Livre), Órgão/Entidade/Empresa (lista controlada). ",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:06:33.999Z",
    id: 75,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação"],
    nomeMetadado: "Entidade (CNPJ)",
    objetivoMetadado:
      "Identificar quais empresas ou órgãos estão sendo tratados na informação.",
    objetivoNaoInformado: false,
    formatoMetadado: "CNPJ",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao: "",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:07:44.184Z",
    id: 76,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: true,
    tipoDocumento: ["REPRESENTAÇÃO", "Informação"],
    nomeMetadado: "Representante (CPF)",
    objetivoMetadado:
      "Identificar o autor pessoa física da representação que foi protocolado ou está sendo analisada.",
    objetivoNaoInformado: false,
    formatoMetadado: "CPF",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: versão inicial atende o metadado do documento mas não atende o de processo.",
    classeProcessual: [
      "103.2.1.2 Processo de análise de representação - Pessoal",
    ],
    imagens: [],
    timestamp: "2025-11-13T17:14:27.287Z",
    id: 77,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: ["REPRESENTAÇÃO", "Informação"],
    nomeMetadado: "Representante (CNPJ)",
    objetivoMetadado:
      "Identificar o autor pessoa jurídica da representação que foi protocolado ou está sendo analisada.",
    objetivoNaoInformado: false,
    formatoMetadado: "CNPJ",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: versão inicial atende o metadado do documento mas não atende o de processo.",
    classeProcessual: [
      "103.2.1.2 Processo de análise de representação - Pessoal",
    ],
    imagens: [],
    timestamp: "2025-11-13T17:14:34.405Z",
    id: 78,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: true,
    tipoDocumento: ["Informação", "DENÚNCIA"],
    nomeMetadado: "Denunciante (CPF)",
    objetivoMetadado:
      "Identificar o autor pessoa física da denúncia que foi protocolado ou está sendo analisada.",
    objetivoNaoInformado: false,
    formatoMetadado: "CPF",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: versão inicial atende o metadado do documento mas não atende o de processo.",
    classeProcessual: ["103.2.1.1 Processo de análise de denúncia - Pessoal"],
    imagens: [],
    timestamp: "2025-11-13T17:17:33.778Z",
    id: 79,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: true,
    tipoDocumento: ["Informação", "DENÚNCIA"],
    nomeMetadado: "Denunciante (CNPJ)",
    objetivoMetadado:
      "Identificar o autor pessoa jurídica da denúncia que foi protocolado ou está sendo analisada.",
    objetivoNaoInformado: false,
    formatoMetadado: "CNPJ",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: versão inicial atende o metadado do documento mas não atende o de processo.",
    classeProcessual: ["103.2.1.1 Processo de análise de denúncia - Pessoal"],
    imagens: [],
    timestamp: "2025-11-13T17:17:59.467Z",
    id: 80,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação"],
    nomeMetadado: "Cargo",
    objetivoMetadado:
      "Identificar os casos em que uma informação trata da análise de um ou mais cargos específicos.",
    objetivoNaoInformado: false,
    formatoMetadado: "Lista controlada",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao: "",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:20:39.955Z",
    id: 81,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação"],
    nomeMetadado: "Carreira",
    objetivoMetadado:
      "Identificar os casos em que uma informação trata da análise de um ou mais carreiras específicas.",
    objetivoNaoInformado: false,
    formatoMetadado: "Lista controlada",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao: "",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:20:58.197Z",
    id: 82,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "Informação",
      "EDITAL",
      "NOTA",
      "NOTA DE INSPEÇÃO",
      "Nota de Auditoria",
      "RELATÓRIO DE INSPEÇÃO",
      "Relatório DE LPA",
      "RELATÓRIO DE MONITORAMENTO",
      "RELATÓRIO FINAL",
      "Relatório FINAL DE AUDITORIA",
      "RELATÓRIO PRÉVIO",
      "Relatório PRÉVIO DE AUDITORIA",
    ],
    nomeMetadado: "Jurisdicionado",
    objetivoMetadado:
      "Identificar os jurisdicionados que estão sendo tratados nos documentos.",
    objetivoNaoInformado: false,
    formatoMetadado: "Lista controlada",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao: "",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:23:07.795Z",
    id: 83,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: true,
    tipoDocumento: ["Informação", "EDITAL"],
    nomeMetadado: "Número do Edital da Seleção",
    objetivoMetadado:
      "Armazenar o número do edital do concurso ou seleção que está sendo examinado.",
    objetivoNaoInformado: false,
    formatoMetadado: "Número de Edital (XX/AAAA)",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Sim",
    observacao:
      "Anotação Fabrício: a versão inicial atende o metadado de documento mas não atende o de processo.",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:28:36.175Z",
    id: 84,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação"],
    nomeMetadado: "Fase da Análise",
    objetivoMetadado:
      "Identificar de qual fase dos processos de denúncia e representação a informação cuida (ex: admissibilidade, mérito, recurso, etc.).",
    objetivoNaoInformado: false,
    formatoMetadado: "Lista controlada",
    admiteMultiplos: false,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao: "",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:31:36.841Z",
    id: 85,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação"],
    nomeMetadado: "Encaminhamento",
    objetivoMetadado:
      "Identificar o encaminhamento proposto pela informação (legalidade, conhecimento, ilegalidade, não conhecimento, etc.).",
    objetivoNaoInformado: false,
    formatoMetadado: "Lista controlada",
    admiteMultiplos: true,
    tipoMetadados: "Simples",
    versaoInicialAtende: "Não",
    observacao: "",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:33:04.592Z",
    id: 86,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: [
      "RELATÓRIO DE INSPEÇÃO",
      "RELATÓRIO PRÉVIO",
      "Relatório PRÉVIO DE AUDITORIA",
      "Relatório FINAL DE AUDITORIA",
      "RELATÓRIO DE MONITORAMENTO",
      "RELATÓRIO FINAL",
    ],
    nomeMetadado: "Encaminhamento do QI/QA",
    objetivoMetadado:
      "Identificar o encaminhamento proposto para cada questão de inspeção (QI) ou questão de auditoria (QA).",
    objetivoNaoInformado: false,
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      "Metadado composto: QI/QA (Texto Livre) e Encaminhamento Proposto (Lista Controlada)",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:35:50.968Z",
    id: 87,
  },
  {
    unidade: "SEFIPE",
    metadadoDeProcesso: false,
    tipoDocumento: ["EXTRATO SIRAC"],
    nomeMetadado: "Ato de Admissão",
    objetivoMetadado:
      "Armazenar os dados do servidor relativo ao ato de admissão analisado.",
    objetivoNaoInformado: false,
    formatoMetadado: "Composto",
    admiteMultiplos: false,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      "Estrutura dos metadados compostos para cada ato de admissão (conforme imagem):\n\na. Edital Normativo: número do edital do concurso relativo ao ato de admissão. (Tipo: número formatado)\nb. Edital de Resultado Final: número do edital do resultado final do concurso relativo ao ato de admissão. (Tipo: número formatado)\nc. Classificação: posição de classificação do candidato admitido. (Tipo: Número Inteiro)\nd. Categoria: categoria na qual foi feita a admissão (Tipo: Lista Controlada - ampla concorrência, PCD, negros, hipossuficientes, etc.)\ne. Indicação: resultado da análise do ato de admissão (Tipo: Lista Controlada - legalidade, conhecimento, ilegalidade, etc.)\n\nf. Servidor/Instituidor: metadados do servidor que foi admitido que originou o direito ao benefício. É obrigatório e admite apenas uma ocorrência.\nf.1. CPF: Cadastro de Pessoa Física do servidor. (Tipo: CPF)\nf.2. Matrícula: Número da matrícula funcional do servidor no órgão. (Tipo: número inteiro)\nf.3. Órgão: Sigla ou nome do órgão ao qual o servidor era vinculado. (Tipo: texto livre)\nf.4. Nome: Nome completo do servidor. (Tipo: texto livre)\nf.5. Cargo: Cargo que o servidor ocupa. (Tipo: texto livre)\n",
    classeProcessual: [],
    imagens: [],
    timestamp: "2025-11-13T17:49:46.841Z",
    id: 88,
  },
  {
    unidade: "NUREC",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação"],
    nomeMetadado: "Encaminhamento do Exame de Admissibilidade",
    objetivoMetadado:
      "Identificar se o recurso foi admitido, qual é a sua espécie e se teve efeito suspensivo e, em caso de proposta de não admissão, qual foi o requisito não atendido.",
    objetivoNaoInformado: false,
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      "Metadado composto: 1) Proposta da admissão (SIM/NÃO); 2) Espécie do Recurso (lista controlada: Recurso de Reconsideração, Pedido de Reexame; Recurso de Revisão; Agravo; Recurso Inominado, etc.). 3) Proposta de efeito suspensivo (SIM/NÃO); 4) Requisito de admissão não atendido (lista controlada que admite mais de um item selecionado: não aplicável, tempestividade; interesse; legitimidade; etc).",
    classeProcessual: [],
    imagens: [],
    tags: [],
    timestamp: "2025-11-28T20:08:28.484Z",
    id: 89,
  },
  {
    unidade: "NUREC",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação"],
    nomeMetadado: "Encaminhamento do Exame de Mérito",
    objetivoMetadado:
      "Identificar o encaminhamento de mérito proposto pelo auditor e as questões que embasaram a proposta de provimento ou provimento parcial de cada recurso.",
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      "Estrutura: 1) encaminhamento proposto (provimento, provimento parcial, desprovimento); 2) razão do encaminhamento proposto (lista controlada: processual - prescrição; processual - outros; material - responsabilização indevida; material - ausência de irregularidade; material - precedente do TCDF e/ou judicial; material - alteração legislativa, etc.).",
    classeProcessual: [],
    timestamp: 1763151340720,
    id: 90,
    objetivoNaoInformado: false,
    precisaRevisao: false,
    estruturaComposta: [
      { nome: "Encaminhamento proposto", formato: "Lista Controlada" },
      { nome: "Razão do encaminhamento", formato: "Lista Controlada" },
    ],
  },
  {
    unidade: "NUREC",
    metadadoDeProcesso: false,
    tipoDocumento: ["Informação"],
    nomeMetadado: "Convergência com a Informação da Fase Recorrida",
    objetivoMetadado:
      "Identificar o alinhamento de entendimento entre o NUREC e a unidade técnica responsável pela informação que subsidiou a decisão que foi objeto de recurso.",
    formatoMetadado: "Composto",
    admiteMultiplos: true,
    tipoMetadados: "Composto",
    versaoInicialAtende: "Não",
    observacao:
      "Composição do metadado: 1) E-doc da Informação considerada (e-doc); 2) Grau de convergência do NUREC (lista controlada: ...)",
    classeProcessual: [],
    timestamp: 1763151340720,
    id: 91,
    objetivoNaoInformado: false,
    precisaRevisao: false,
    estruturaComposta: [
      { nome: "E-doc da Informação", formato: "Texto" },
      { nome: "Grau de convergência", formato: "Lista Controlada" },
    ],
  },
];

const initialDataIds = new Set(initialData.map((d) => d.id));

// --- Listas Controladas para o Formulário ---
const UNIDADES_TCDF = [
  "SEGEDAM – Secretaria-Geral de Administração",
  "SECOF – Secretaria-Geral de Contabilidade, Orçamento e Finanças",
  "SEORC – Serviço de Execução Orçamentária",
  "SEFIN – Serviço de Execução Financeira",
  "SECON – Serviço de Contabilidade",
  "SELIP – Secretaria-Geral de Licitação, Material e Patrimônio",
  "SELIC – Serviço de Licitação",
  "SERCO – Serviço de Contratos",
  "SEMAP – Serviço de Material e Patrimônio",
  "SEGEP – Secretaria-Geral de Gestão de Pessoas",
  "SELEG – Serviço de Legislação de Pessoal",
  "SECAF – Serviço de Cadastro Funcional",
  "SEPAG – Serviço de Pagamento de Pessoal",
  "SEGED – Serviço de Gestão de Desempenho e de Desenvolvimento de Competências",
  "SESAP – Secretaria-Geral de Engenharia e Serviços de Apoio",
  "SEMAN – Serviço de Manutenção",
  "SEPROJ – Serviço de Obras e Projetos",
  "SESOP – Serviço de Segurança e Suporte Operacional",
  "SETRA – Serviço de Transportes",
  "SESBE – Secretaria-Geral de Saúde, Qualidade de Vida e Bem-Estar",
  "DSAUD – Divisão de Assistência Direta à Saúde",
  "DIBEM – Divisão de Qualidade de Vida e Bem-Estar",
  "SESOC – Serviço de Saúde, Segurança Ocupacional e Qualidade de Vida",
  "SEGEDOC – Secretaria-Geral de Gestão de Documentos e da Informação",
  "SEPROT – Serviço de Protocolo e Publicações Oficiais",
  "SEDIGI – Serviço de Documentos Digitais e Proteção de Dados",
  "SEARQ – Serviço de Arquivo e Gestão de Documentos",
  "SEMEMO – Serviço de Memória Institucional",
];

const FORMATOS_DADO = [
  "Texto livre",
  "CNPJ",
  "Booleano (SIM/NÃO)",
  "Data",
  "Data/Hora",
  "E-doc",
  "Lista controlada",
  "Número de Processo SEI",
  "Número de Contrato (XX/AAAA)",
  "Número (inteiro)",
  "Valor (R$)",
  "E-mail",
  "URL",
  "Número de Processo Judicial",
  "Número de Edital (XX/AAAA)",
  "CPF",
  "Composto",
  "Número de Processo TCDF",
  "Número de Processo de Barramento",
].sort();

const CLASSES_PROCESSUAIS_TCDF = [
  "1.1.1.1 Processo relativo a reestruturação administrativa",
  "1.1.2.1 Processo relativo à proposição de normas",
  "1.1.2.1 Processo de revisão de normas",
  "Processo relativo a convênio, termo de cooperação, acordo, protocolo de intenções e termo de parceria",
  "1.2.1.1 Processo referente à elaboração do plano plurianual",
  "1.2.2.1 Processo referente ao plano estratégico",
  "1.2.3.1 Processo de Elaboração e acompanhamento da execução do Plano Geral de Ação",
  "1.2.4.1 Relatórios Atividades do TCDF para remessa à CLDF",
  "1.2.4.2 Manuais de Rotinas e Procedimentos Administrativos",
  "1.3.1.1 Processo de monitoramento ao sistema de controle interno",
  "1.3.1.2 Processo de comunicação com Órgãos de Fiscalização Externa",
  "1.3.1.3 Processo de gestão e acompanhamento do Plano de Ação Anual da DCI",
  "1.3.1.4 Processo de gestão e acompanhamento dos Indicadores de Controle Interno",
  "1.3.1.5 Processo de criação ou atualização de Instruções Normativas do SCI",
  "1.3.1.6 Processo de criação ou atualização dos Manuais de Procedimentos Administrativos",
  "1.3.1.7 Processo de acompanhamento dos macrocontroles dos sistemas administrativos",
  "1.3.2.1 Processo de análise da Tomada de Contas Anual do TCDF",
  "1.3.2.2 Processo de análise dos Relatórios de Gestão Fiscal",
  "1.3.2.3 Processo de acompanhamento do calendário de obrigações",
  "1.3.2.4 Processo de análise de legalidade dos atos de admissão de pessoal",
  "1.3.2.5 Processo de análise da legalidade dos atos de concessão de abono permanência, aposentadoria e pensão",
  "1.3.3.1 Processo de realização de auditoria",
  "1.3.3.2 Processo de realização de inspeção",
  "1.4.1.1 Relatório de atividades trimestral",
  "1.4.1.2 Processo de acompanhamento de ação judicial",
  "1.4.1.3 Processo de atendimento de requisições, ordens judiciais ou administrativas",
  "1.4.2.1 Despacho Normativo",
  "1.4.2.2 Parecer sobre Questões Jurídicas e Judiciais",
  "1.4.2.3 Parecer Técnico",
  "1.4.2.4 Súmula",
  "1.5.1.1 Processo relativo à política de acesso à informação do TCDF",
  "1.5.2.1 Memorando de encaminhamento de denúncia",
  "1.5.2.2 Memorando de encaminhamento de elogio",
  "1.5.2.3 Memorando de encaminhamento de reclamação",
  "1.5.2.4 Memorando de encaminhamento de sugestão",
  "1.5.2.5 Memorando de encaminhamento de solicitação de informação",
  "1.5.2.6 Memorando de encaminhamento de solicitação de diversa",
  "1.5.2.7 Ofício de encaminhamento de demandas a outros órgãos",
  "1.5.2.8 Ofício de resposta ao requerente",
  "1.5.3.1 Dossiê de acompanhamento do pedido de acesso à informação",
  "1.5.3.2 Ofício de resposta a pedido de acesso a informação",
  "1.5.3.3 Processo relativo a pedido de cópia e vista de processo",
  "1.5.3.4 Processo relativo a resposta a pedido informação",
  "1.6.1.1 Processo relativo à auditoria de Controle Externo",
  "1.7.1.1 Processo de análise de solicitação de informação",
  "1.7.1.2 Processo de análise de esclarecimento de informação",
  "1.8.1.1 Processo relativo ao relatório de atividades de corregedoria",
  "1.8.1.2 Processo relativo ao plano de correição",
  "1.8.1.3 Processo de correição geral ordinária",
  "1.8.1.4 Processo de correição geral extraordinária",
  "1.8.1.5 Processo de inspeção ordinária",
  "2.1.1.1 Processo referente ao programa de recrutamento e seleção",
  "2.1.1.2 Processo referente ao Programa de alocação e integração",
  "2.1.1.3 Processo de diagnóstico e planejamento/dimensionamento da força de trabalho",
  "2.1.2.1 Processo relativo ao plano de desenvolvimento de competências",
  "2.1.2.2 Processo relativo ao sistema de gestão de desempenho competente",
  "2.1.2.3 Processo relativo ao plano bianual de capacitação",
  "2.1.2.4 Processo relativo ao plano de formação e desenvolvimento para gerenciar",
  "2.1.2.5 Processo relativo ao programa de desenvolvimento funcional",
  "2.1.2.6 Processo relativo ao projeto pedagógico institucional de educação corporativa",
  "2.1.2.7 Processo de diagnóstico de necessidades de capacitação e treinamento",
  "2.1.3.1 Processo referente ao Plano de Carreira, Cargos e Remunerações",
  "2.1.3.2 Processo referente ao Programa de Benefícios",
  "2.1.4.1 Processo referente ao levantamento de necessidades de capacitação – LNC",
  "2.1.4.2 Processo referente à avaliação do impacto das ações de capacitação",
  "2.1.4.3 Processo referente à consultoria interna em gestão de pessoas",
  "2.1.4.4 Processo referente à comunicação interna e endomarketing",
  "2.1.4.5 Processo referente ao programa de gestão do clima organizacional",
  "2.1.5.1 Processo relativo ao banco de talentos",
  "2.1.5.2 Processo relativo ao sistema de cadastro e folha de pagamento",
  "2.1.6.1 Processo relativo ao programa de assistência à saúde – PROSAÚDE",
  "2.1.6.2 Processo relativo ao programa de incentivo à cultura – PROCULTURA",
  "2.1.6.3 Processo relativo ao programa de gestão da qualidade de vida no trabalho",
  "2.1.6.4 Processo relativo ao programa de preparação para a aposentadoria e programa de apoio ao aposentado",
  "2.1.7.1 Processo relativo às políticas de remuneração",
  "2.1.7.2 Processo relativo à criação, transformação e reavaliação de cargos e funções",
  "2.1.7.3 Processo relativo às atribuições dos cargos",
  "2.1.7.4 Processo relativo ao remanejamento de cargos",
  "2.1.8.1 Processo de estudos especiais relativo a cargos e funções",
  "2.1.8.2 Processo de estudos especiais relativo a gestão da base cadastral",
  "2.1.8.3 Processo de estudos especiais relativo a direitos, benefícios e vantagens",
  "2.1.8.4 Processo de estudos especiais relativo a gestão da base financeira",
  "2.1.8.5 Processo de estudos especiais relativo a gestão de competências e desempenho",
  "2.1.8.6 Processo de estudos especiais relativo a educação corporativa",
  "2.1.8.7 Processo de estudos especiais relativo ao regime disciplinar",
  "2.1.8.8 Processo de estudos especiais relativo ao gerenciamento de estágios",
  "2.1.9.1 Processo de proposição de normas",
  "2.1.9.2 Processo de revisão de normas",
  "2.1.10.1 Processo relativo ao Regime Geral de Previdência Social",
  "2.1.10.2 Processo relativo ao Regime Próprio de Previdência Social",
  "2.1.10.3 Processo relativo ao Regime de Previdência Complementar",
  "2.2.1.1 Processo de realização de concurso público para provimento de cargos",
  "2.2.1.2 Processo de recurso administrativo contra resultado de concurso público para provimento de cargos",
  "2.2.1.3 Processo de recurso judicial contra resultado de concurso público para provimento de cargos",
  "2.2.1.4 Processo relativo a irregularidades ocorridas em concurso público para provimento de cargos",
  "2.2.2.1 Processo de vacância de cargo efetivo",
  "2.2.2.2 Processo de exoneração de cargo efetivo",
  "2.2.2.3 Processo relativo a posse tornada sem efeito",
  "2.2.2.4 Processo relativo à desistência de posse em cargo público",
  "2.2.2.5 Processo relativo à acumulação de cargos e rendimentos públicos",
  "2.2.2.6 Processo relativo à reversão de cargo público",
  "2.2.2.7 Processo relativo ao aproveitamento de cargo público",
  "2.2.2.8 Processo relativo à reintegração de cargo público",
  "2.2.2.9 Processo relativo à acumulação de cargo público com atividade empresarial",
  "2.2.2.10 Processo relativo à acumulação de rendimentos/proventos e submissão ao teto remuneratório",
  "2.2.3.1 Processo de designação para função de confiança",
  "2.2.3.2 Processo de nomeação para cargo em comissão",
  "2.2.3.3 Processo de vacância de cargo em comissão",
  "2.2.3.4 Processo de designação de servidores para substituição",
  "2.2.3.5 Processo de alteração de vínculo funcional",
  "2.2.4.1 Processo relativo a requerimento judicial de reenquadramento",
  "2.2.4.2 Processo relativo a requerimento judicial de isenção de IRPF de servidor ativo",
  "2.2.4.3 Processo relativo a desconto judicial de pensão alimentícia",
  "2.2.5.1 Memorando de movimentação interna de pessoal",
  "2.2.5.2 Processo de cessão de servidor",
  "2.2.5.3 Processo de requisição de servidor",
  "2.3.1.1 Dossiê de dados funcionais de Conselheiro",
  "2.3.1.2 Dossiê de dados funcionais de Auditor",
  "2.3.1.3 Dossiê de dados funcionais de Membros do Ministério Público",
  "2.3.1.4 Dossiê de dados Funcionais de Servidores",
  "2.3.2.1 Processo de emissão de identidade funcional de servidores",
  "2.3.2.2 Processo de emissão de identidade funcional de membros",
  "2.3.2.3 Processo relativo à emissão de crachás de servidores",
  "2.3.2.4 Processo relativo à emissão de crachá de membros",
  "2.3.3.1 Registro de Frequência de servidores",
  "2.3.4.1 Processo de recadastramento de dados de servidor inativo",
  "2.3.4.2 Processo de recadastramento de dados de membro inativo",
  "2.3.4.3 Processo de recadastramento de pensionista",
  "2.3.4.4 Processo de declaração anual de bens",
  "2.3.4.5 Processo de recadastramento de dados de servidor ativo",
  "2.3.4.6 Processo de recadastramento de dados de membro ativo",
  "2.3.4.7 Processo de comprovação da condição de estudante relativo ao programa de assistência à saúde (Pró-Saúde)",
  "2.3.4.8 Processo de comprovação da condição de dependência econômica relativo ao programa de assistência à saúde (Pró-Saúde)",
  "2.3.5.1 Recibo de expediente",
  "2.3.6.1 Processo relativo à emissão de certidão funcional",
  "2.3.6.2 Processo relativo à emissão de certidão de tempo de contribuição",
  "2.3.6.3 Processo relativo à emissão de declaração funcional",
  "2.3.6.4 Processo relativo à emissão de declaração comprobatória",
  "2.3.7.1 Processo relativo a elogio funcional",
  "2.3.7.2 Processo relativo a denúncia de servidor",
  "2.4.1.1 Processo de concessão de adicional de Insalubridade",
  "2.4.1.2 Processo de concessão de adicional de Periculosidade",
  "2.4.1.3 Processo de concessão de adicional por Serviço Extraordinário",
  "2.4.1.4 Processo de concessão de adicional noturno",
  "2.4.1.5 Processo de concessão de gratificação por trabalhos com raios X ou substâncias radioativas",
  "2.4.2.1 Processo de concessão de adicional de tempo de serviço",
  "2.4.2.2 Processo de concessão de adicional de qualificação",
  "2.4.2.3 Processo relativo à incorporação de vantagem pessoal",
  "2.4.2.4 Processo relativo a vantagens pessoais nominalmente identificáveis",
  "2.4.3.1 Processo de concessão de auxílio natalidade",
  "2.4.3.2 Processo de concessão de auxílio pré-escolar",
  "2.4.3.3 Processo de concessão de auxílio funeral",
  "2.4.3.4 Processo de concessão de gratificação por encargo de curso ou concurso",
  "2.4.4.1 Processo de concessão de diária e passagem para viagem",
  "2.4.4.2 Processo de concessão de auxílio-transporte",
  "2.4.4.3 Processo de concessão de auxílio-alimentação",
  "2.4.4.4 Processo de concessão de abono pecuniário",
  "2.4.4.5 Processo de concessão de auxílio-moradia",
  "2.4.4.6 Processo de concessão Indenização de Despesa com Serviços de Comunicação",
  "2.4.4.7 Processo relativo a assistência médica (Pró-Saúde)",
  "2.4.4.8 Processo de concessão de abono de permanência",
  "2.4.4.9 Processo de concessão de indenização de férias",
  "2.4.5.1 Processo de concessão de licença por motivo de afastamento do cônjuge ou companheiro",
  "2.4.5.2 Processo de concessão de licença por motivo de doença em pessoa da família",
  "2.4.5.3 Processo de concessão de licença para o serviço militar",
  "2.4.5.4 Processo de concessão de licença para atividade política",
  "2.4.5.5 Processo de concessão de licença prêmio por assiduidade",
  "2.4.5.6 Processo de concessão de licença para tratar de interesses particulares",
  "2.4.5.7 Processo de concessão de licença para desempenho de mandato classista",
  "2.4.5.8 Processo de concessão de licença paternidade",
  "2.4.5.9 Processo de concessão de licença maternidade",
  "2.4.5.10 Processo de concessão de licença por motivo de adoção de filhos",
  "2.4.5.11 Processo de concessão de licença por motivo de acidente de serviço",
  "2.4.5.12 Processo de concessão de abono de ponto",
  "2.4.6.1 Processo de concessão de afastamento para servir em outro órgão ou entidade",
  "2.4.6.2 Processo de concessão para exercício em outro órgão",
  "2.4.6.3 Processo de concessão de afastamento exercício de mandato eletivo",
  "2.4.6.4 Processo de concessão de afastamento para estudo ou missão no exterior",
  "2.4.6.5 Processo de concessão de afastamento para participar de competição desportiva",
  "2.4.6.6 Processo de concessão de afastamento para frequência em curso de formação",
  "2.4.6.7 Processo de concessão de horário especial temporário a servidor com deficiência",
  "2.4.6.8 Processo de concessão de horário especial temporário a servidor que tenha cônjuge, filho ou dependente com deficiência",
  "2.4.6.9 Processo de concessão de horário especial temporário a servidor matriculado em curso da educação básica e da educação superior",
  "2.4.6.10 Processo de concessão de horário especial temporário a servidora lactante",
  "2.4.6.11 Processo de concessão de horário especial temporário a servidor atleta",
  "2.4.6.12 Processo de concessão de afastamento para participar de programa de pós-graduação stricto sensu",
  "2.4.8.1 Processo de escala de férias",
  "2.4.8.2 Processo de recesso regimental",
  "2.4.9.1 Processo de aposentadoria",
  "2.4.9.2 Processo de aposentadoria por invalidez",
  "2.4.9.3 Processo de aposentadoria especial",
  "2.4.9.4 Processo de pensão civil",
  "2.4.9.5 Processo de averbação de tempo de contribuição",
  "2.4.9.6 Processo de apuração de tempo previdenciário",
  "2.4.9.1 Processo de adesão ao Regime de Previdência Complementar (DFPREVICOM)",
  "2.5.1.1 Dossiê de dados financeiros de servidores",
  "2.5.1.2 Dossiê de dados financeiros de membros",
  "2.5.1.3 Contracheques",
  "2.5.1.4 Processo de inclusão de dependentes para fins de imposto de renda",
  "2.5.1.5 Processo relativo às folhas de pagamento de inativos e pensionistas",
  "2.5.2.1 Dossiê de Reembolso",
  "2.5.2.2 Processo de Reembolso Anual",
  "2.5.2.3 Processo de reembolso parcial de atendimento domiciliar",
  "2.5.3.1 Processo relativo a acerto financeiro decorrente de férias e décimo terceiro salário",
  "2.5.3.2 Processo relativo a acerto financeiro decorrente de exoneração",
  "2.5.3.3 Processo relativo a acerto financeiro decorrente de aposentadoria",
  "2.5.3.4 Processo relativo à devolução de valores descontados em folha de pagamento",
  "2.5.3.5 Processo relativo a acerto financeiro decorrente de gozo de licença sem vencimentos",
  "2.5.3.6 Processo relativo a acerto financeiro decorrente de falecimento de servidor ativo, inativo ou pensionista",
  "2.5.3.7 Processo relativo a acerto financeiro decorrente de exoneração ou dispensa de cargo em comissão ou função de confiança",
  "2.5.3.8 Processo relativo a acerto financeiro decorrente de retorno de servidor cedido à origem",
  "2.5.3.9 Processo relativo a acerto financeiro decorrente de limite do teto remuneratório",
  "2.5.4.1 Processo de consignação de empréstimos em folha de pagamento",
  "2.5.4.2 Processo de consignação de pecúlios em folha de pagamento",
  "2.5.4.3 Processo de consignação de contribuições sindicais em folha de pagamento",
  "2.5.4.4 Processo de desconto de valores em folha de pagamento",
  "2.5.5.1 Processo de ressarcimento de valores decorrentes de cessão de servidor",
  "2.5.5.2 Processo de devolução de valores a outros órgãos",
  "2.6.1.1 Processo de identificação de perfis ocupacionais",
  "2.6.1.2 Processo de mapeamento de competências",
  "2.6.1.3 Processo de estudos de desenvolvimento de competências",
  "2.6.2.1 Processo relativo ao programa de desenvolvimento profissional",
  "2.6.2.2 Processo relativo aos planos de ações de avaliação de desempenho e desenvolvimento",
  "2.6.2.3 Processo anual de gestão do desempenho competente",
  "2.6.2.4 Processo de avaliação de estágio probatório",
  "2.6.2.5 Dossiê de avaliação de desempenho de servidores requisitados",
  "2.6.2.6 Processo relativo à pesquisa de clima organizacional",
  "2.6.2.7 Processo do programa de preparação para a aposentadoria",
  "2.6.2.8 Processo do programa de qualidade de vida no trabalho",
  "2.6.3.1 Processo de progressão funcional",
  "2.6.3.2 Processo de promoção funcional",
  "2.7.1.1 Processo de capacitação profissional de membros e servidores realizada pelo TCDF",
  "2.7.1.2 Processo de capacitação profissional de membros e servidores realizada por outras institutions",
  "2.7.1.3 Processo de promoção de cursos de pós-graduação pelo TCDF",
  "2.7.1.4 Processo de concessão de bolsas de estudos para cursos de pós-graduação",
  "2.7.1.5 Processo de concessão de bolsas de estudo para cursos de graduação",
  "2.7.1.6 Processo de concessão de bolsas de estudo para cursos de idioma estrangeiro",
  "2.7.1.7 Processo relativo ao programa de instrutoria interna",
  "2.7.1.8 Processo relativo ao programa de formação de novos servidores",
  "2.7.2.1 Processo de ações pedagógicas direcionadas aos jurisdicionados",
  "2.7.2.2 Processo de ações pedagógicas direcionadas à sociedade",
  "2.7.2.3 Processo relativo à organização de concursos de trabalhos científicos",
  "2.8.1.1 Processo de sindicância",
  "2.8.1.2 Processo de sindicância patrimonial",
  "2.8.1.3 Processo administrativo disciplinar",
  "2.8.1.4 Processo relativo a abandono de cargo",
  "2.8.1.5 Processo relativo a inassiduidade habitual",
  "2.8.1.6 Processo relativo a apuração de irregularidade",
  "2.9.1.1 Processo relativo ao programa de estágio do TCDF",
  "2.9.2.1 Dossiês de estagiários remunerados",
  "2.9.2.2 Dossiês de estagiários não remunerados",
  "2.10.1.1 Processo relativo aos programas de saúde e bem-estar",
  "2.10.1.2 Processo de cooperação médica com outras institutions",
  "2.10.2.1 Prontuário médico",
  "2.10.2.2 Prontuário odontológico",
  "2.10.2.3 Agenda de atendimento médico",
  "2.10.2.4 Agenda atendimento psicológico",
  "2.10.2.5 Agenda de atendimento odontológico",
  "2.10.3.1 Processo relativo a acidente de trabalho",
  "2.10.3.2 Boletim de licenças médicas",
  "3.1.1.1 Processo relativo à cooperação técnica entre instituições",
  "3.1.1.2 Processo relativo aos planos e programas de TI",
  "3.1.2.1 Processo de proposição de normas",
  "3.1.2.2 Processo de revisão de normas",
  "3.2.1.1 Formulário para credenciamento de usuários externos para uso de sistema do TCDF",
  "3.2.1.2 Formulário para credenciamento de usuários internos para uso de sistemas de terceiros",
  "3.3.1.1 Processo relativo ao desenvolvimento e manutenção de sistemas",
  "3.3.1.2 Processo de registro de propriedade de software",
  "3.3.1.3 Processo relativo à documentação de desenvolvimento de sistemas",
  "3.3.2.1 Processo relativo à manutenção da infraestrutura de rede",
  "3.3.2.2 Processo relativo ao serviço de impressão",
  "3.3.3.1 Processo relativo ao às atividades de atendimento aos usuários",
  "4.1.1.1 Relatório de atividades",
  "4.1.1.2 Processo relacionado às estatísticas de custo e consumo",
  "4.1.2.1 Processo de proposição de normas",
  "4.1.2.2 Processo de revisão de normas",
  "4.2.1.1 Processo relativo aos trabalhos de Comissão Permanente ou Especial de Licitação",
  "4.2.2.1 Processo de assinatura inicial de instrumento contratual",
  "4.2.2.2 Processo de aditamento contratual",
  "4.2.2.3 Processo de repactuação contratual",
  "4.2.2.4 Processo de reajuste contratual",
  "4.2.2.5 Processo de reequilíbrio econômico",
  "4.2.2.6 Processo de prorrogação de vigência contratual",
  "4.2.2.7 Processo de aplicação de penalidade",
  "4.2.2.8 Processo de rescisão do contrato",
  "4.2.2.9 Processo de emissão de atestado de capacidade técnica",
  "4.2.2.10 Processo de abertura e acompanhamento de conta vinculada",
  "4.2.3.1 Processo de aquisição de material de consumo",
  "4.2.3.2 Processo de aquisição de bens patrimoniais móveis",
  "4.2.3.3 Processo de aquisição de bens patrimoniais imóveis",
  "4.2.3.4 Processo de aquisição de bens materiais e patrimoniais via suprimento de fundos",
  "4.2.3.5 Processo de aquisição de bens materiais e patrimoniais via registro de preços",
  "4.2.4.1 Processo de contratação de obras e reformas",
  "4.2.4.2 Processo de contratação de serviço técnico profissional",
  "4.2.4.3 Processo de contratação de serviço de operação de sistema de som",
  "4.2.4.4 Processo de contratação de prestação de serviço com locação de mão de obra",
  "4.2.4.5 Processo de contratação de prestação de serviço com locação de mão de obra e fornecimento de material",
  "4.2.4.6 Processo de contratação de serviços complementares",
  "4.2.4.7 Processo de contratação de serviços de utilidade pública",
  "4.2.4.8 Processo de contratação de empresa especializada na área de saúde",
  "4.2.4.9 Processo de aquisição de serviços via suprimento de fundos",
  "4.2.4.10 Processo de cessão, autorização e permissão de uso",
  "4.2.4.11 Atestado de recebimento de serviços",
  "4.2.5.1 Processo relativo a estudos e projetos de engenharia e arquitetura",
  "4.2.5.2 Processo relativo à alteração de layout do ambiente de trabalho",
  "4.2.5.3 Processo relativo à sinalização visual",
  "4.2.6.1 Processo relativo ao uso de espaço físico institucional",
  "4.3.1.1 Processo relativo ao inventário físico de bens patrimoniais móveis e imóveis",
  "4.3.1.2 Processo relativo ao inventário físico-financeiro de bens patrimoniais",
  "4.3.1.3 Termos de guarda e responsabilidade - TGR",
  "4.3.2.1 Processo de reposição de bens patrimoniais móveis e imóveis",
  "4.3.2.2 Processo de movimentação interna de bens patrimoniais",
  "4.3.2.3 Termo de movimentação de bens patrimoniais - TMBP",
  "4.3.3.1 Processo acerca da utilização de imóvel funcional",
  "4.3.4.1 Processo de contratação de seguro para bens patrimoniais móveis",
  "4.3.4.2 Processo de contratação de seguro para bens patrimoniais imóveis",
  "4.3.5.1 Processo de locação de máquinas e equipamentos",
  "4.3.5.2 Processo de locação de imóvel",
  "4.3.5.3 Processo de locação de bens móveis",
  "4.3.6.1 Processo de alienação de bens patrimoniais",
  "4.3.6.2 Processo de baixa de bens patrimoniais",
  "4.4.1.1 Processo relativo ao inventário físico de material de consumo",
  "4.4.1.2 Processo relativo ao inventário físico-financeiro de material de consumo",
  "4.4.1.3 Boletim de saída de material de consumo",
  "4.4.1.4 Sistema de controle de material de consumo",
  "4.4.1.5 Termos de responsabilidade",
  "4.4.1.6 Termos de recebimento de material de consumo",
  "4.4.2.1 Atestado de recebimento de material de consumo",
  "4.4.2.2 Processo relativo aos trabalhos de Comissão de Recebimento de Material de Consumo",
  "4.4.3.1 Processo de doação de material de consumo",
  "4.4.3.2 Processo de baixa de material de consumo inservível",
  "4.5.1.1 Processo de renovação da frota de veículos",
  "4.5.1.2 Processo de aquisição de veículos",
  "4.5.1.3 Processo para incorporação de veículos doados na frota",
  "4.5.2.1 Processo de terceirização da frota de veículos de serviço",
  "4.5.2.2 Processo de terceirização da frota de veículos de representação oficial",
  "4.5.2.3 Processo de locação de veículos eventuais",
  "4.5.3.1 Processo de contratação de seguro para veículos oficiais",
  "4.5.3.2 Processo relativo a ocorrência de furto de veículo oficial",
  "4.5.4.1 Processo para confecção de placas oficiais para veículos de representação",
  "4.5.4.2 Processo de autorização para servidor dirigir veículos oficiais",
  "4.5.4.3 Processo relativo a acidente de trânsito",
  "4.5.4.4 Processo relativo a multa de trânsito",
  "4.5.4.5 Processo referente ao uso irregular de veículos oficiais",
  "4.5.4.6 Relatório de percurso",
  "4.5.5.1 Processo de contratação de empresa para fornecimento de combustível",
  "4.5.5.2 Relatórios de Consumo de Combustível",
  "4.5.6.1 Processo de doação de veículos",
  "4.5.6.2 Processo de devolução de veículos",
  "5.1.1.1 Processo de proposição e revisão de normas",
  "5.1.1.2 Processo relativo à política interna de gestão contábil, orçamentária e financeira",
  "5.2.1.1 Processo de elaboração de proposta e alterações da Lei de Diretrizes Orçamentárias",
  "5.2.1.2 Processo de elaboração e revisão do Plano Plurianual",
  "5.2.1.3 Processo de elaboração da Proposta Orçamentária",
  "5.2.2.1 Processo de execução orçamentária",
  "5.2.3.1 Processo de Gastos com Publicidade e Propaganda",
  "5.2.3.2 Processo de Publicação do Extrato de Compras",
  "5.2.3.3 Memorando referente ao Extrato de Compras",
  "5.3.1.1 Processo de conciliação bancária",
  "5.3.2.1 Processo de repasses de superávit financeiro",
  "5.3.3.1 Processo de pagamento referente à aquisição de material de consumo",
  "5.3.3.2 Processo de pagamento referente à aquisição de bens patrimoniais móveis",
  "5.3.3.3 Processo de pagamento referente à aquisição de bens patrimoniais imóveis",
  "5.3.3.4 Processo de pagamento referente à aquisição de bens materiais e patrimoniais via suprimento de fundos",
  "5.3.3.5 Processo de pagamento referente à aquisição de bens materiais e patrimoniais via registro de preços",
  "5.3.4.1 Processo de pagamento referente à contratação de obras e reformas",
  "5.3.4.2 Processo de pagamento referente à contratação de serviços técnicos profissionais",
  "5.3.4.3 Processo de pagamento referente à contratação de serviço de operação de sistema de som",
  "5.3.4.4 Processo de pagamento referente à contratação de prestação de serviço com locação de mão de obra",
  "5.3.4.5 Processo de pagamento referente à contratação de prestação de serviço com locação de mão de obra e fornecimento de material",
  "5.3.4.6 Processo de pagamento referente à contratação de serviços complementares",
  "5.3.4.7 Processo de pagamento referente à contratação de serviços de utilidade pública",
  "5.3.4.8 Processo de pagamento referente à contratação de empresa especializada na área de saúde",
  "5.3.4.9 Processo de pagamento referente à contratação de serviços de TI",
  "5.3.4.10 Processo de pagamento referente à contratação de serviços via suprimento de fundos",
  "5.3.5.1 Processo de pagamento da remuneração dos servidores",
  "5.3.5.2 Processo de pagamento de contribuições previdenciárias",
  "5.3.5.3 Processo de pagamento de ressarcimento de despesas de pessoal cedido ao TCDF",
  "5.4.1.1 Processo de Tomada de Contas do TCDF",
  "5.4.2.1 Processo de relatório de gestão fiscal quadrimestral do TCDF",
  "5.4.2.2 Processo de relatório de gestão fiscal mensal do TCDF",
  "5.4.3.1 Processo de concessão, aplicação e comprovação de suprimentos de fundos",
  "5.4.4.1 Processo de conciliação de material de consumo",
  "5.4.4.2 Processo de conciliação de bens patrimoniais",
  "5.4.5.1 Processo de declaração de retenção de ISS na fonte",
  "5.4.5.2 Processo de declaração de débitos e créditos tributários federais",
  "5.4.5.3 Processo de declaração de imposto de renda retido na fonte",
  "6.1.1.1 Processo relativo à política de gestão da biblioteca",
  "6.1.1.2 Processo relativo à cooperação entre bibliotecas",
  "6.1.1.3 Processo relativo aos instrumentos de organização e gestão da informação",
  "6.1.2.1 Processo relativo à política de gestão documentos",
  "6.1.2.2 Processo relativo aos instrumentos de gestão arquivística",
  "6.1.2.3 Processo relativo à Comissão Permanente de Avaliação de Documentos",
  "6.1.3.1 Processo relativo à política de jurisprudência",
  "6.1.4.1 Processo de proposição de normas",
  "6.1.4.2 Processo de revisão de normas",
  "6.1.5.1 Processo de proposição de normas",
  "6.1.5.2 Processo de revisão de normas",
  "6.1.6.1 Processo de proposição de normas",
  "6.1.6.2 Processo de revisão de normas",
  "6.2.1.1 Processo relativo à composição do Conselho Editorial",
  "6.2.1.2 Processo relativo à confecção da revista do TCDF",
  "6.2.2.1 Processo relativo à manutenção do sistema de segurança eletromagnético e automação da biblioteca",
  "6.2.3.1 Processo relativo à organização do acervo bibliográfico",
  "6.2.3.2 Processo relativo à doação e permuta de material bibliográfico",
  "6.2.3.3 Processo relativo ao descarte de livros",
  "6.2.3.4 Processo de avaliação do acervo bibliográfico",
  "6.2.3.5 Processo referente à digitalização do acervo",
  "6.2.3.6 Processo referente à microfilmagem do acervo",
  "6.2.4.1 Processo relativo à assinatura de periódicos",
  "6.2.4.2 Processo relativo à renovação de assinatura de periódicos",
  "6.2.5.1 Processo relativo a empréstimo permanente",
  "6.2.5.2 Processo relativo a empréstimo entre bibliotecas",
  "6.2.6.1 Processo relativo à encadernação de livros e periódicos",
  "6.3.1.1 Guia de remessa interna de documentos",
  "6.3.1.2 Guia de remessa de malote",
  "6.3.1.3 Processo de Comunicação Via Barramento PEN",
  "6.3.2.1 Processo de assistência técnica",
  "6.3.3.1 Processo relativo à guarda terceirizada de documentos",
  "6.3.3.2 Recibo de coleta/implantação/empréstimo de caixas",
  "6.3.4.1 Processo de eliminação de documentos",
  "6.3.4.2 Processo de transferência de documentos",
  "6.3.4.3 Processo de recolhimento de documentos",
  "6.3.5.1 Processo relativo à conservação e preservação de documentos",
  "6.4.1.1 Boletim de decisões do TCDF",
  "6.4.1.2 Boletim de decisões judiciais",
  "6.4.1.3 Boletim “inovações legislativas”",
  "6.4.1.4 Boletim de decisões normativas",
  "6.4.1.5 Boletim de Consultas ao TCDF",
  "6.4.1.6 Boletim de legislação de interesse",
  "6.4.1.7 Boletim relativo à Lei de Licitação e Contratos",
  "6.4.1.8 Boletim de Súmulas",
  "6.4.1.9 Boletim de pareceres normativos da PGDF",
  "7.1.1.1 Processo relativo à política de comunicação institucional do TCDF",
  "7.1.1.2 Processo relativo ao Plano anual de Publicidade e Propaganda",
  "7.1.1.3 Processo relativo à Rede de Comunicação dos Tribunais de Contas",
  "7.1.2.1 Artigo, nota e notícia",
  "7.1.2.2 Credencial de jornalista",
  "7.1.2.3 Pauta da imprensa",
  "7.1.2.4 Release e sinopse",
  "7.1.2.5 Site institucional",
  "7.1.3.1 Banco de imagens",
  "7.1.3.2 Registro sonoro",
  "7.1.3.3 Vídeo institucional",
  "7.1.4.1 Clipping",
  "7.1.5.1 E-mail de solicitação de publicação de notícia",
  "7.1.5.2 Memorando de solicitação de publicação de notícia",
  "7.2.1.1 Processo relativo a visita oficial",
  "7.2.1.2 Dossiê de evento",
  "7.2.1.3 Memorando de solicitação de disponibilização de objetos e funcionários",
  "7.2.2.1 Comunicado de luto oficial",
  "7.2.2.2 Ofício de agradecimento, cumprimento, despedida ou pêsames",
  "7.2.2.3 Ofício encaminhando o programa e o cerimonial da solenidade ou recepção",
  "7.2.2.4 Convites para eventos",
  "7.3.1.1 Processo relativo à identidade visual do TCDF",
  "7.3.1.2 Processo relativo à campanha institucional",
  "7.3.1.3 Processo relativo à contratação de agências de publicidade",
  "8.1.1.1 Processo relativo à política de gestão das atividades de plenário",
  "8.1.2.1 Processo de proposição de normas",
  "8.1.2.2 Processo de revisão de normas",
  "8.2.1.1 Roteiro de sessão plenária",
  "8.2.1.2 Pauta de sessão plenária",
  "8.2.1.3 Ata de sessão plenária",
  "8.2.2.1 Recibo de ofício de envio de documentos para publicação",
  "8.2.3.1 E-mail de solicitação de inclusão de legislação no SASP",
  "8.2.3.2 E-mail de solicitação de material bibliográfico",
  "8.2.4.1 Recibos de Ofícios GP",
  "8.2.4.2 Recibos de Comunicados de Audiência",
  "8.2.4.3 Recibos de Citação",
  "8.2.5.1 Livro de decisões",
  "9.1.1.1 Processo referente aos planos e programas do Ministério Público de Contas",
  "9.1.1.2 Processo relativo a convênio e acordo de cooperação técnica",
  "9.1.2.1 Processo de proposição de normas",
  "9.1.2.2 Processo de revisão de normas",
  "9.1.3.1 Processo de estudos especiais de comissões e grupos de trabalho",
  "9.2.1.1 Processo referente ao Plano Estratégico",
  "9.2.2.1 Relatórios de Atividades Trimestral do MPCDF",
  "9.2.2.2 Relatórios de Atividades Anual do MPCDF",
  "9.2.2.3 Manuais de Rotinas e Procedimentos Administrativos",
  "9.2.2.4 Boletim Eletrônico",
  "9.2.2.5 Relatório Executivo",
  "9.3.1.1 Processo relativo à política de acesso à informação do MPCDF",
  "9.3.1.2 Carta de Serviço ao Cidadão",
  "9.3.2.1 Memorando de encaminhamento de denúncia",
  "9.3.2.2 Memorando de encaminhamento de denúncia anônima",
  "9.3.2.3 Memorando de encaminhamento de elogio",
  "9.3.2.4 Memorando de encaminhamento de reclamação",
  "9.3.2.5 Memorando de encaminhamento de sugestão",
  "9.3.2.6 Memorando de encaminhamento de solicitação",
  "9.3.2.7 Ofício de encaminhamento de demandas a outros órgãos",
  "9.3.2.8 Ofício de resposta ao requerente",
  "9.3.3.1 Dossiê de acompanhamento do pedido de acesso à informação",
  "9.3.3.2 Ofício de resposta a pedido de acesso a informação",
  "9.3.3.3 Processo relativo a resposta a pedido informação",
  "9.4.1.1 Clipping",
  "9.5.1.1 Processo de Procedimento Interno de avaliação de estádio probatório de Procurador",
];

const TIPOS_DOCUMENTO_TCDF = [
  "ACORDO",
  "ACORDO DE COOPERAÇÃO",
  "ACÓRDÃO",
  "AGENDA",
  "ALEGAÇÕES",
  "ALVARÁ",
  "ANAIS",
  "ANEXO",
  "ANEXO DE PROCESSO GDF",
  "ANOTAÇÃO",
  "ANTEPROJETO",
  "ANÁLISE",
  "APARTADO",
  "APOSTILAMENTO",
  "APRESENTAÇÃO",
  "APÓLICE",
  "ATA DE PROCEDIMENTO LICITATÓRIO",
  "ATA DE REGISTRO DE PREÇOS",
  "ATA DE REUNIÃO",
  "ATA DE SESSÃO",
  "ATA DE SESSÃO DE CONTAS DE GOVERNO",
  "ATESTADO",
  "ATESTO",
  "ATO DE INSTAURAÇÃO",
  "ATO NORMATIVO",
  "ATO SUJEITO A REGISTRO",
  "AUTO",
  "AUTORIZAÇÃO DE INSPEÇÃO",
  "AVISO",
  "AVISO DE RECEBIMENTO",
  "BALANCETE",
  "BALANÇO FINANCEIRO",
  "BALANÇO ORÇAMENTÁRIO",
  "BALANÇO PATRIMONIAL",
  "BILHETE",
  "BOLETIM",
  "BOLETO",
  "CALENDÁRIO",
  "CAPTURA DE TELA",
  "CARTA",
  "CARTAZ",
  "CARTEIRA",
  "CARTÃO",
  "CERTIDÃO",
  "CERTIFICADO",
  "CERTIFICADO DE AUDITORIA",
  "CHECK-LIST",
  "CHEQUE",
  "CIENTIFICAÇÃO",
  "CITAÇÃO",
  "CLASSIFICAÇÃO CONTÁBIL",
  "CNH",
  "CNPJ",
  "COMPLEMENTO DE PROPOSTA DE DECISÃO",
  "COMPROVANTE",
  "COMPROVANTE DE ADIMPLEMENTO DE OBRIGAÇÕES TRIBUTÁRIAS",
  "COMPROVANTE DE CUMPRIMENTO DE OBRIGAÇÕES TRABALHISTAS",
  "COMPROVANTE DE PAGAMENTO",
  "COMUNICADO",
  "COMUNICAÇÃO DE AUDIÊNCIA",
  "CONCILIAÇÃO BANCÁRIA",
  "CONJUNTO PROBATÓRIO",
  "CONSULTA",
  "CONTA",
  "CONTRACHEQUE",
  "CONTRARRAZÕES",
  "CONTRATO",
  "CONVENÇÃO",
  "CONVITE",
  "CONVÊNIO",
  "CORRESPONDÊNCIA REGISTRADA",
  "COTA",
  "CPF/CIC",
  "CRACHÁ",
  "CREDENCIAL",
  "CREDENCIAMENTO - TCDF SAÚDE",
  "CRONOGRAMA",
  "CROQUI",
  "CURRÍCULO",
  "CÉDULA",
  "CÓPIA DE PROCESSO",
  "DADOS BANCÁRIOS",
  "DEBÊNTURE",
  "DECISÃO",
  "DECISÃO DA PRESIDÊNCIA",
  "DECISÃO LIMINAR",
  "DECISÃO NORMATIVA",
  "DECLARAÇÃO",
  "DECLARAÇÃO CONJUNTA DOS SALDOS DE EMPENHOS",
  "DECLARAÇÃO DA INEXISTÊNCIA DE NEPOTISMO",
  "DECLARAÇÃO DE VOTO",
  "DECRETO",
  "DEFESA",
  "DEFESA PRÉVIA",
  "DELIBERAÇÃO",
  "DEMONSTRATIVO",
  "DEMONSTRATIVO DA DÍVIDA FLUTUANTE",
  "DEMONSTRATIVO DA EXECUÇÃO DA DESPESA POR PROGRAMA",
  "DEMONSTRATIVO DA RECEITA ORÇADA E ARRECADADA",
  "DEMONSTRATIVO DAS TOMADAS DE CONTAS ESPECIAIS",
  "DEMONSTRATIVO DE ATUALIZAÇÃO DE VALORES - SINDEC",
  "DEMONSTRATIVO DE CONTRATO DE PROGRAMA",
  "DEMONSTRATIVO DE CONTRATO DE RATEIO",
  "DEMONSTRATIVO DE CRÉDITOS ADICIONAIS",
  "DEMONSTRATIVO DE CRÉDITOS ORÇAMENTÁRIOS - LOA",
  "DEMONSTRATIVO DE DESPESAS COM RECURSOS DE CONTRATO DE RATEIO",
  "DEMONSTRATIVO DE REGISTRO",
  "DEMONSTRATIVO DE RESTOS A PAGAR NÃO PROCESSADOS",
  "DEMONSTRATIVO DE RESTOS A PAGAR PROCESSADOS",
  "DEMONSTRATIVO DE SALDO ORÇAMENTÁRIO",
  "DEMONSTRATIVO DE SUPRIMENTO DE FUNDOS",
  "DEMONSTRATIVO DO PLANO DE APLICAÇÃO DE RECURSOS",
  "DEMONSTRATIVO FINANCEIRO",
  "DEMONSTRAÇÃO",
  "DEMONSTRAÇÃO DAS MUTAÇÕES DO PATRIMÔNIO LÍQUIDO",
  "DEMONSTRAÇÃO DAS VARIAÇÕES PATRIMONIAIS",
  "DEMONSTRAÇÃO DO FLUXO DE CAIXA",
  "DEMONSTRAÇÃO DO RESULTADO DO EXERCÍCIO",
  "DENÚNCIA",
  "DEPOIMENTO",
  "DESIGNAÇÃO",
  "DESPACHO",
  "DESPACHO DE REEMBOLSO DE BOLSA DE ESTUDOS",
  "DESPACHO SINGULAR",
  "DETALHAMENTO DE BENEFÍCIOS",
  "DIAGNÓSTICO",
  "DIPLOMA",
  "DIRETRIZ",
  "DISSERTAÇÃO",
  "DISTRIBUIÇÃO DE PROCESSOS - TERMO",
  "DIÁRIO",
  "DOCUMENTO COMPROBATÓRIO",
  "DOCUMENTO DE ALTERAÇÃO OU EXTINÇÃO DE CONSÓRCIO PÚBLICO",
  "DOCUMENTO DE AUDITORIA",
  "DOCUMENTO E-CONTAS",
  "DOCUMENTO PARTICULAR",
  "DOCUMENTO PESSOAL COMPROBATÓRIO",
  "DOCUMENTOS COMPLEMENTARES",
  "DOCUMENTOS DA OUVIDORIA",
  "DOCUMENTOS NÃO ENCAMINHADOS PELO GESTOR",
  "DOCUMENTOS SISAUDIT",
  "DOD - DOCUMENTO DE OFICIALIZAÇÃO DA DEMANDA",
  "DOSSIÊ",
  "EDITAL",
  "EMAIL",
  "EMBARGOS",
  "EMBARGOS DE DECLARAÇÃO",
  "EMENDA REGIMENTAL",
  "ENCAMINHAMENTO DE PORTARIA",
  "ERRATA DE CIENTIFICAÇÃO",
  "ERRATA DE CITAÇÃO",
  "ERRATA DE COMUNICAÇÃO DE AUDIÊNCIA",
  "ERRATA DE DECISÃO",
  "ERRATA DE NOTIFICAÇÃO",
  "ESCALA",
  "ESCLARECIMENTO",
  "ESCRITURA",
  "ESCRITURAÇÃO",
  "ESPECIFICAÇÃO DE REQUISITOS",
  "ESTATUTO",
  "ESTRATÉGIA",
  "ESTUDO",
  "ETIQUETA",
  "EXAME",
  "EXPOSIÇÃO DE MOTIVOS",
  "EXTRATO DE COMPRAS",
  "EXTRATO DE PUBLICIDADE",
  "EXTRATO SIRAC",
  "FICHA 4ª ICE",
  "FLUXOGRAMA",
  "FOLHA",
  "FOLHETO/FOLDER",
  "FORMULÁRIO",
  "FORMULÁRIO DE AVALIAÇÃO DE ANTEPROJETO",
  "FORMULÁRIO DE PAGAMENTO",
  "FORMULÁRIO DE PUBLICAÇÃO DE SUPRIMENTO DE FUNDOS",
  "FOTOGRAFIA",
  "GRADE CURRICULAR",
  "GUIA",
  "GUIA DE REMESSA",
  "HISTÓRICO",
  "IDENTIFICAÇÃO DO ROL DE RESPONSÁVEIS",
  "IMPUGNAÇÃO",
  "INDICAÇÃO",
  "INFORMAÇÃO",
  "INFORMAÇÃO DE PAPELETA",
  "INFORMAÇÃO ORÇAMENTÁRIA FINANCEIRA",
  "INFORME",
  "INSTRUMENTO DE MEDIÇÃO DE RESULTADO",
  "INSTRUÇÃO",
  "INSTRUÇÃO NORMATIVA",
  "INTENÇÃO",
  "JURISPRUDÊNCIA SELECIONADA",
  "LAUDO DE AMOSTRA DE LICITAÇÃO",
  "LAUDO MÉDICO",
  "LEI ORÇAMENTÁRIA",
  "LICENÇA",
  "LISTA/LISTAGEM",
  "LIVRO",
  "MANDADO JUDICIAL",
  "MANIFESTAÇÃO",
  "MANIFESTO",
  "MANUAL",
  "MAPA",
  "MATERIAL",
  "MATRIZ DE ACHADOS",
  "MATRIZ DE AVALIAÇÃO DE RISCOS",
  "MATRIZ DE MONITORAMENTO",
  "MATRIZ DE PLANEJAMENTO",
  "MATRIZ DE PLANEJAMENTO DE MONITORAMENTO",
  "MATRIZ DE RESPONSABILIZAÇÃO",
  "MATÉRIA",
  "MEDIDA PROVISÓRIA",
  "MEMORANDO",
  "MEMORANDO-CIRCULAR",
  "MEMORANDO-CONJUNTO",
  "MEMORIAL",
  "MEMÓRIA",
  "MENSAGEM",
  "MINUTA",
  "MOVIMENTAÇÃO",
  "MOÇÃO",
  "NORMA",
  "NOTA",
  "NOTA DE AUDITORIA",
  "NOTA DE INSPEÇÃO",
  "NOTA DE LEVANTAMENTO",
  "NOTA DE MONITORAMENTO",
  "NOTA DE TRANSCRIÇÃO",
  "NOTA FISCAL / FATURA",
  "NOTA TÉCNICA",
  "NOTAS EXPLICATIVAS DAS DEMONSTRAÇÕES CONTÁBEIS",
  "NOTIFICAÇÃO",
  "OFÍCIO",
  "OFÍCIO CIRCULAR",
  "OFÍCIO DE COMUNICAÇÃO",
  "OFÍCIO GP",
  "OFÍCIO GP CIRCULAR",
  "OFÍCIO-CONJUNTO",
  "ORDEM BANCÁRIA",
  "ORDEM DE FORNECIMENTO",
  "ORDEM DE SERVIÇO",
  "ORGANOGRAMA",
  "ORIENTAÇÃO",
  "ORÇAMENTO",
  "OUTROS",
  "PANFLETO",
  "PAPEL DE TRABALHO - PT",
  "PAPELETA DE BENEFÍCIOS",
  "PARECER",
  "PARECER DE AUDITORIA INTERNA",
  "PARECER DO CONSELHO DE ADMINISTRAÇÃO/ÓRGÃO EQUIVALENTE",
  "PARECER MÉDICO",
  "PASSAPORTE",
  "PAUTA",
  "PEDIDO",
  "PEDIDO DE ACESSO À INFORMAÇÃO",
  "PEDIDO DE RECONSIDERAÇÃO",
  "PEDIDO DE SUSTENTAÇÃO ORAL",
  "PESQUISA DE PREÇOS",
  "PETIÇÃO",
  "PLANILHA",
  "PLANO",
  "PLANO DE ENSINO",
  "PLANTA",
  "PORTARIA",
  "PRECATÓRIO",
  "PROCESSO (EXCLUÍDOS PROCESSO TCDF E GDF)",
  "PROCESSO GDF",
  "PROCURAÇÃO",
  "PROGRAMA",
  "PROJETO BÁSICO / TERMO DE REFERÊNCIA",
  "PROJETO DE PARECER PRÉVIO - RAPP",
  "PRONTUÁRIO",
  "PRONUNCIAMENTO",
  "PROPOSTA",
  "PROSPECTO",
  "PROTOCOLO ELETRÔNICO",
  "PROVA",
  "PUBLICAÇÃO",
  "PUBLICAÇÃO DO(S) ACORDÃO(S)",
  "PÁGINA",
  "QUADRO",
  "QUESTIONÁRIO",
  "RAZÕES DE JUSTIFICATIVA",
  "RECEITA",
  "RECIBO DE DOCUMENTO",
  "RECIBO DE EXPEDIENTE",
  "RECLAMAÇÃO",
  "RECOMENDAÇÃO",
  "RECURSO",
  "REFERENDO",
  "REGIMENTO",
  "REGISTROS MENTORH",
  "REGISTROS SIRAC",
  "REGULAMENTO",
  "RELATÓRIO",
  "RELATÓRIO / PROPOSTA DE DECISÃO",
  "RELATÓRIO / VOTO",
  "RELATÓRIO ANALÍTICO - VERSÃO FINAL",
  "RELATÓRIO ANALÍTICO - VERSÃO PRELIMINAR",
  "RELATÓRIO ANALÍTICO / PARECER PRÉVIO",
  "RELATÓRIO CONCLUSIVO DO ORGANIZADOR DAS CONTAS",
  "RELATÓRIO DA COMISSÃO DE INVENTÁRIO DO ALMOXARIFADO",
  "RELATÓRIO DA DIRETORIA",
  "RELATÓRIO DE ANÁLISE TÉCNICA DE PPP/CONCESSÕES",
  "RELATÓRIO DE ATIVIDADES DO GESTOR",
  "RELATÓRIO DE AUDITORIA",
  "RELATÓRIO DE AUDITORIA CONTROLE INTERNO",
  "RELATÓRIO DE AUDITORIA INDEPENDENTE",
  "RELATÓRIO DE CÁLCULO DE PRESCRIÇÃO",
  "RELATÓRIO DE INSPEÇÃO",
  "RELATÓRIO DE LEVANTAMENTO",
  "RELATÓRIO DE LPA",
  "RELATÓRIO DE LPM",
  "RELATÓRIO DE MONITORAMENTO",
  "RELATÓRIO DO ÓRGÃO CENTRAL DE CONTABILIDADE",
  "RELATÓRIO FINAL",
  "RELATÓRIO FINAL DE AUDITORIA",
  "RELATÓRIO PRÉVIO",
  "RELATÓRIO PRÉVIO DE AUDITORIA",
  "RELATÓRIOS DE INVENTÁRIO PATRIMONIAL DA ADMINISTRAÇÃO DIRETA",
  "RELATÓRIOS DE INVENTÁRIO PATRIMONIAL DA ADMINISTRAÇÃO INDIRETA",
  "RELAÇÃO",
  "RELAÇÃO DE AUXÍLIOS, SUBVENÇÕES E CONTRIBUIÇÕES",
  "RELEASE",
  "REPRESENTAÇÃO",
  "REPRESENTAÇÃO CONJUNTA",
  "REPRESENTAÇÃO DO MPC",
  "REQUERIMENTO",
  "REQUISIÇÃO",
  "RESERVA ORÇAMENTÁRIA",
  "RESOLUÇÃO",
  "RESPOSTA DE NOTA DE AUDITORIA",
  "RESULTADO",
  "RESUMO",
  "REVISTA",
  "RG",
  "ROTEIRO",
  "SENTENÇA",
  "SINOPSE",
  "SOLICITAÇÃO",
  "SÚMULA",
  "TABELA",
  "TELEGRAMA",
  "TERMO",
  "TERMO ADITIVO",
  "TERMO CIRCUNSTANCIADO DE REGULARIZAÇÃO",
  "TERMO DE ACEITE",
  "TERMO DE ANEXAÇÃO",
  "TERMO DE APENSAÇÃO",
  "TERMO DE ARQUIVAMENTO",
  "TERMO DE AUTUAÇÃO",
  "TERMO DE AVOCAÇÃO DE PROCESSO",
  "TERMO DE COOPERAÇÃO",
  "TERMO DE DEPOIMENTO",
  "TERMO DE DESANEXAÇÃO",
  "TERMO DE DESAPENSAÇÃO",
  "TERMO DE DESARQUIVAMENTO",
  "TERMO DE DESIGNAÇÃO",
  "TERMO DE EMBARGO",
  "TERMO DE ENCERRAMENTO DE TRAMITAÇÃO FÍSICA DE PROCESSO",
  "TERMO DE JULGAMENTO LICITATÓRIO",
  "TERMO DE MOVIMENTAÇÃO DE BENS PATRIMONIAIS",
  "TERMO DE NOTIFICAÇÃO",
  "TERMO DE NÃO IMPEDIMENTO",
  "TERMO DE POSSE",
  "TERMO DE REAJUSTE",
  "TERMO DE RETIRADA",
  "TERMO DE SOLICITAÇÃO DE DIGITALIZAÇÃO",
  "TESE",
  "TESTAMENTO",
  "TOMADA DE CIÊNCIA",
  "TÍTULO DE PENSÃO",
  "VOLUME",
  "VOLUME DE PROCESSO GDF",
  "VOTO DE DESEMPATE",
  "VOTO VISTA / MANIFESTAÇÃO DE VISTA",
  "VÍDEO",
  "ÁUDIO",
];

const FORMATOS_SUB_CAMPO = [
  "Texto",
  "Número",
  "Data",
  "Lista Controlada",
].sort();

// --- COMPONENTES AUXILIARES ---
const KPIBox = ({ icon: Icon, title, value, color }) => {
  const colors = {
    blue: {
      bg: "bg-blue-50",
      text: "text-blue-600",
      border: "border-l-blue-500",
    },
    indigo: {
      bg: "bg-indigo-50",
      text: "text-indigo-600",
      border: "border-l-indigo-500",
    },
    amber: {
      bg: "bg-amber-50",
      text: "text-amber-600",
      border: "border-l-amber-500",
    },
    emerald: {
      bg: "bg-emerald-50",
      text: "text-emerald-600",
      border: "border-l-emerald-500",
    },
  };
  const c = colors[color] || colors.blue;
  return (
    <div
      className={`bg-white p-5 rounded-xl border border-slate-200 shadow-sm flex items-center space-x-4 border-l-4 ${c.border}`}
    >
      <div className={`${c.bg} p-3 rounded-lg ${c.text}`}>
        <Icon className="h-6 w-6" />
      </div>
      <div>
        <p className="text-sm font-medium text-slate-500">{title}</p>
        <h3 className="text-2xl font-bold text-slate-900">{value}</h3>
      </div>
    </div>
  );
};

const ExportButton = ({ data }) => {
  const headers = [
    { label: "ID", key: "id" },
    { label: "Nome do Metadado", key: "nomeMetadado" },
    { label: "Unidade", key: "unidade" },
    { label: "Objetivo", key: "objetivoMetadado" },
    { label: "Estrutura", key: "tipoMetadados" },
    { label: "Formato", key: "formatoMetadado" },
    { label: "Admite Múltiplos", key: "admiteMultiplos" },
    { label: "É de Processo", key: "metadadoDeProcesso" },
    { label: "Classes Processuais", key: "classeProcessual" },
    { label: "Tipos de Documento", key: "tipoDocumento" },
    { label: "Finalidade Documento", key: "finalidadeTipoDocumento" },
    { label: "Estrutura Composta", key: "estruturaComposta" },
    { label: "Observação", key: "observacao" },
  ];

  const handleExport = () => {
    const processed = data.map((item) => ({
      ...item,
      admiteMultiplos: item.admiteMultiplos ? "Sim" : "Não",
      metadadoDeProcesso:
        item.metadadoDeProcesso === true || item.metadadoDeProcesso === "Sim"
          ? "Sim"
          : "Não",
      tipoDocumento: Array.isArray(item.tipoDocumento)
        ? item.tipoDocumento.join("; ")
        : "",
      classeProcessual: Array.isArray(item.classeProcessual)
        ? item.classeProcessual.join("; ")
        : item.classeProcessual || "",
      finalidadeTipoDocumento: item.finalidadeTipoDocumento || "",
      estruturaComposta: Array.isArray(item.estruturaComposta)
        ? item.estruturaComposta
            .map((c) => `${c.nome} (${c.formato})`)
            .join("; ")
        : "",
    }));

    const headerRow = headers.map((h) => `"${h.label}"`).join(",");
    const rows = processed.map((item) =>
      headers
        .map((h) => {
          let val = item[h.key];
          if (val === undefined || val === null) val = "";
          return `"${String(val).replace(/"/g, '""')}"`;
        })
        .join(",")
    );

    const csvContent = [headerRow, ...rows].join("\n");
    const blob = new Blob([csvContent], { type: "text/csv;charset=utf-8;" });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.setAttribute("href", url);
    link.setAttribute("download", "metadados-tcdf.csv");
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  };

  return (
    <button
      onClick={handleExport}
      className="flex items-center gap-2 px-4 py-2.5 bg-emerald-600 hover:bg-emerald-700 text-white rounded-lg font-bold shadow-sm transition-colors text-sm"
    >
      <Download className="h-4 w-4" /> Exportar CSV
    </button>
  );
};

const ConfirmationModal = ({ isOpen, onClose, onConfirm, title, message }) => {
  if (!isOpen) return null;
  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-slate-900/60 backdrop-blur-sm animate-in fade-in">
      <div className="relative bg-white w-full max-w-md m-8 p-6 rounded-xl shadow-2xl flex flex-col">
        <h2 className="text-lg font-bold text-slate-800">{title}</h2>
        <p className="text-sm text-slate-600 mt-2 mb-6">{message}</p>
        <div className="flex justify-end gap-3">
          <button
            onClick={onClose}
            className="px-4 py-2 border border-slate-300 rounded-lg text-slate-700 font-bold hover:bg-slate-50"
          >
            Cancelar
          </button>
          <button
            onClick={onConfirm}
            className="px-4 py-2 bg-red-600 hover:bg-red-700 text-white rounded-lg font-bold shadow-md"
          >
            Confirmar Exclusão
          </button>
        </div>
      </div>
    </div>
  );
};

// --- LÓGICA DE VALIDAÇÃO (ZOD) ---
function validarCPF(cpf) {
  const cpfLimpo = String(cpf).replace(/\D/g, "");
  if (cpfLimpo.length !== 11 || /^(\d)\1+$/.test(cpfLimpo)) return false;
  let soma = 0;
  for (let i = 0; i < 9; i++) soma += parseInt(cpfLimpo.charAt(i)) * (10 - i);
  let resto = 11 - (soma % 11);
  let dv1 = resto === 10 || resto === 11 ? 0 : resto;
  if (dv1 !== parseInt(cpfLimpo.charAt(9))) return false;
  soma = 0;
  for (let i = 0; i < 10; i++) soma += parseInt(cpfLimpo.charAt(i)) * (11 - i);
  resto = 11 - (soma % 11);
  let dv2 = resto === 10 || resto === 11 ? 0 : resto;
  return dv2 === parseInt(cpfLimpo.charAt(10));
}

function validarCNPJ(cnpj) {
  const cnpjLimpo = String(cnpj).replace(/\D/g, "");
  if (cnpjLimpo.length !== 14 || /^(\d)\1+$/.test(cnpjLimpo)) return false;
  let tamanho = 12,
    soma = 0,
    pos = 5;
  for (let i = 0; i < tamanho; i++) {
    soma += parseInt(cnpjLimpo.charAt(i)) * pos--;
    if (pos < 2) pos = 9;
  }
  let resultado = soma % 11 < 2 ? 0 : 11 - (soma % 11);
  if (resultado !== parseInt(cnpjLimpo.charAt(12))) return false;
  tamanho = 13;
  soma = 0;
  pos = 6;
  for (let i = 0; i < tamanho; i++) {
    soma += parseInt(cnpjLimpo.charAt(i)) * pos--;
    if (pos < 2) pos = 9;
  }
  resultado = soma % 11 < 2 ? 0 : 11 - (soma % 11);
  return resultado === parseInt(cnpjLimpo.charAt(13));
}

const formSchema = z
  .object({
    unidade: z.string().min(1, "A unidade é obrigatória."),
    nomeMetadado: z.string().min(3, "O nome do metadado é obrigatório."),
    objetivoMetadado: z.string().optional(),
    objetivoNaoInformado: z.boolean().optional(),
    tipoMetadados: z.enum(["Simples", "Composto"]),
    formatoMetadado: z.string().min(1, "O formato é obrigatório."),
    exemploDeValor: z.string().optional(),
    admiteMultiplos: z.enum(["Sim", "Não"]),
    metadadoDeProcesso: z.enum(["Sim", "Não"]),
    tipoDocumento: z
      .union([z.array(z.string()), z.string(), z.boolean(), z.null()])
      .transform((val) => {
        if (Array.isArray(val)) return val;
        if (typeof val === "string" && val.trim() !== "") return [val];
        return [];
      })
      .refine((arr) => arr.length > 0, {
        message: "Selecione ao menos um tipo de documento.",
      }),
    finalidadeTipoDocumento: z
      .string()
      .min(1, "A finalidade do tipo documental é obrigatória."),
    classeProcessual: z
      .union([z.array(z.string()), z.string(), z.boolean(), z.null()])
      .transform((val) => {
        if (Array.isArray(val)) return val;
        if (typeof val === "string" && val.trim() !== "") return [val];
        return [];
      })
      .optional(),
    observacao: z.string().optional(),
    estruturaComposta: z
      .array(
        z.object({
          nome: z.string().optional(),
          formato: z.string().optional(),
        })
      )
      .optional(),
  })
  .superRefine((data, ctx) => {
    if (
      !data.objetivoNaoInformado &&
      (!data.objetivoMetadado || data.objetivoMetadado.trim() === "")
    ) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message:
          "O objetivo é obrigatório, a menos que seja marcado como 'não informado'.",
        path: ["objetivoMetadado"],
      });
    }
    if (data.tipoMetadados === "Composto") {
      if (!data.estruturaComposta || data.estruturaComposta.length === 0) {
        ctx.addIssue({
          code: z.ZodIssueCode.custom,
          message: "Para metadados compostos, defina pelo menos um sub-campo.",
          path: ["estruturaComposta", "root"],
        });
      } else {
        data.estruturaComposta.forEach((campo, index) => {
          if (!campo.nome || campo.nome.trim() === "") {
            ctx.addIssue({
              code: z.ZodIssueCode.custom,
              message: "O nome do sub-campo é obrigatório.",
              path: ["estruturaComposta", index, "nome"],
            });
          }
          if (!campo.formato || campo.formato.trim() === "") {
            ctx.addIssue({
              code: z.ZodIssueCode.custom,
              message: "O formato do sub-campo é obrigatório.",
              path: ["estruturaComposta", index, "formato"],
            });
          }
        });
      }
    }
    if (
      data.formatoMetadado === "CPF" &&
      data.exemploDeValor &&
      !validarCPF(data.exemploDeValor)
    ) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: "CPF inválido.",
        path: ["exemploDeValor"],
      });
    }
    if (
      data.formatoMetadado === "CNPJ" &&
      data.exemploDeValor &&
      !validarCNPJ(data.exemploDeValor)
    ) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: "CNPJ inválido.",
        path: ["exemploDeValor"],
      });
    }
    if (
      data.formatoMetadado === "Número de Processo SEI" &&
      data.exemploDeValor &&
      !/^\d{5}\.\d{6}\/\d{4}-\d{2}$/.test(data.exemploDeValor)
    ) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: "Formato do processo SEI inválido (NNNNN.NNNNNN/AAAA-DD).",
        path: ["exemploDeValor"],
      });
    }
  });

const MetadataForm = ({
  onSubmit,
  onCancel,
  isSubmitting,
  existingData,
  showToast,
}) => {
  const [filtroClasse, setFiltroClasse] = useState("");
  const [filtroDoc, setFiltroDoc] = useState("");
  const {
    register,
    handleSubmit,
    watch,
    control,
    formState: { errors },
    reset,
  } = useForm({
    resolver: zodResolver(formSchema),
    defaultValues: {
      unidade: "",
      nomeMetadado: "",
      objetivoMetadado: "",
      objetivoNaoInformado: false,
      tipoMetadados: "Simples",
      metadadoDeProcesso: "Não",
      formatoMetadado: "Texto livre",
      admiteMultiplos: "Não",
      tipoDocumento: [],
      finalidadeTipoDocumento: "",
      classeProcessual: [],
      observacao: "",
      exemploDeValor: "",
      estruturaComposta: [{ nome: "", formato: "Texto" }],
    },
  });

  useEffect(() => {
    if (existingData) {
      let defaultClasses = [];
      if (Array.isArray(existingData.classeProcessual))
        defaultClasses = existingData.classeProcessual;
      else if (
        existingData.classeProcessual &&
        existingData.classeProcessual !== "Não informado"
      )
        defaultClasses = [existingData.classeProcessual];

      let defaultDocs = [];
      if (Array.isArray(existingData.tipoDocumento))
        defaultDocs = existingData.tipoDocumento;
      else if (existingData.tipoDocumento)
        defaultDocs = [existingData.tipoDocumento];

      reset({
        ...existingData,
        tipoDocumento: defaultDocs,
        classeProcessual: defaultClasses,
        finalidadeTipoDocumento: existingData.finalidadeTipoDocumento || "",
        admiteMultiplos: existingData.admiteMultiplos ? "Sim" : "Não",
        metadadoDeProcesso:
          existingData.metadadoDeProcesso === true ||
          existingData.metadadoDeProcesso === "Sim"
            ? "Sim"
            : "Não",
      });
    }
  }, [existingData, reset]);

  const { fields, append, remove } = useFieldArray({
    control,
    name: "estruturaComposta",
  });

  const tipoMetadados = watch("tipoMetadados");
  const objetivoNaoInformado = watch("objetivoNaoInformado");
  const formatoMetadado = watch("formatoMetadado");
  const watchedClasseProcessual = watch("classeProcessual") || [];
  const watchedTipoDocumento = watch("tipoDocumento") || [];

  const formatoOpcoes = FORMATOS_DADO;
  const showExemplo = ["CPF", "CNPJ", "Número de Processo SEI"].includes(
    formatoMetadado
  );

  const docsFiltrados = useMemo(
    () =>
      TIPOS_DOCUMENTO_TCDF.filter((td) =>
        td.toLowerCase().includes(filtroDoc.toLowerCase())
      ),
    [filtroDoc]
  );

  const classesFiltradas = useMemo(
    () =>
      CLASSES_PROCESSUAIS_TCDF.filter((cp) =>
        cp.toLowerCase().includes(filtroClasse.toLowerCase())
      ),
    [filtroClasse]
  );

  const handleFinalSubmit = (data) => {
    onSubmit(data, existingData ? existingData.id : null);
  };

  const handleValidationErrors = (errors) => {
    console.log("Erros de validação:", errors);
    if (showToast)
      showToast(
        "Preencha todos os campos obrigatórios destacados a vermelho.",
        true
      );
  };

  return (
    <div className="max-w-4xl mx-auto">
      <div className="bg-white rounded-xl shadow-sm border border-slate-200 overflow-hidden relative">
        <div className="bg-slate-50 border-b border-slate-200 px-8 py-5 flex items-center gap-3">
          <div className="bg-blue-100 p-2 rounded-lg text-blue-700">
            <PlusCircle className="h-6 w-6" />
          </div>
          <div>
            <h2 className="text-xl font-bold text-slate-800">
              {existingData ? "Editar Metadado" : "Novo Metadado"}
            </h2>
            <p className="text-sm text-slate-500">
              {existingData
                ? "Altere os dados abaixo para atualizar o registo."
                : "Preencha os campos abaixo para registar a necessidade de um novo metadado e indicar os tipos de documento aos quais ele deverá ser vinculado."}
            </p>
          </div>
        </div>

        <form
          onSubmit={handleSubmit(handleFinalSubmit, handleValidationErrors)}
          className="flex flex-col"
        >
          <div className="p-8 space-y-8 flex-1">
            <div className="bg-blue-50 border-l-4 border-blue-500 p-4 rounded-r-lg shadow-sm">
              <p className="text-sm text-blue-800 font-medium flex items-center gap-2">
                <Info className="h-5 w-5 flex-shrink-0" />
                <span>
                  <strong>Aviso:</strong> Neste momento, o e-TCDF permite apenas
                  o cadastro de metadados simples (não compostos) para
                  documentos. Ainda não está disponível a implementação de
                  metadados para processos; entretanto, as unidades já podem
                  indicar as suas necessidades para futura incorporação.
                </span>
              </p>
            </div>

            {/* Seção 1 */}
            <div>
              <h3 className="text-sm font-bold text-slate-800 uppercase tracking-wider mb-4 pb-2 border-b border-slate-200">
                1. Identificação Principal
              </h3>
              <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">
                    Unidade Responsável <span className="text-red-500">*</span>
                  </label>
                  <select
                    {...register("unidade")}
                    className={`w-full px-4 py-2 border rounded-lg bg-slate-50 ${
                      errors.unidade ? "border-red-500" : "border-slate-300"
                    }`}
                  >
                    <option value="">Selecione...</option>
                    {UNIDADES_TCDF.map((u) => (
                      <option key={u} value={u}>
                        {u}
                      </option>
                    ))}
                    <option value="Outra">Outra</option>
                  </select>
                  {errors.unidade && (
                    <p className="text-xs text-red-600 mt-1">
                      {errors.unidade.message}
                    </p>
                  )}
                </div>

                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">
                    Nome do Metadado <span className="text-red-500">*</span>
                  </label>
                  <input
                    type="text"
                    {...register("nomeMetadado")}
                    placeholder="Ex: Número do Processo SEI, Data da Irregularidade..."
                    className={`w-full px-4 py-2 border rounded-lg ${
                      errors.nomeMetadado
                        ? "border-red-500"
                        : "border-slate-300"
                    }`}
                  />
                  {errors.nomeMetadado && (
                    <p className="text-xs text-red-600 mt-1">
                      {errors.nomeMetadado.message}
                    </p>
                  )}
                </div>

                <div className="md:col-span-2">
                  <label className="block text-sm font-medium text-slate-700 mb-1">
                    Objetivo / Descrição <span className="text-red-500">*</span>
                  </label>
                  <textarea
                    {...register("objetivoMetadado")}
                    disabled={objetivoNaoInformado}
                    rows="3"
                    placeholder="Ex: Identificação do número de processos judiciais mencionados nos documentos encaminhados..."
                    className={`w-full px-4 py-2 border rounded-lg disabled:bg-slate-100 ${
                      errors.objetivoMetadado
                        ? "border-red-500"
                        : "border-slate-300"
                    }`}
                  ></textarea>
                  {errors.objetivoMetadado && (
                    <p className="text-xs text-red-600 mt-1">
                      {errors.objetivoMetadado.message}
                    </p>
                  )}
                  <div className="mt-2 flex items-center gap-2">
                    <input
                      type="checkbox"
                      id="objetivoNaoInformado"
                      {...register("objetivoNaoInformado")}
                      className="h-4 w-4 text-blue-600 border-gray-300 rounded cursor-pointer"
                    />
                    <label
                      htmlFor="objetivoNaoInformado"
                      className="text-sm text-slate-600 font-medium cursor-pointer select-none"
                    >
                      Objetivo não informado
                    </label>
                  </div>
                </div>
              </div>
            </div>

            {/* Seção 2 */}
            <div>
              <h3 className="text-sm font-bold text-slate-800 uppercase tracking-wider mb-4 pb-2 border-b border-slate-200">
                2. Estrutura e Formatação
              </h3>
              <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">
                    Estrutura (Tipo) <span className="text-red-500">*</span>
                  </label>
                  <select
                    {...register("tipoMetadados")}
                    className="w-full px-4 py-2 border border-slate-300 rounded-lg bg-slate-50"
                  >
                    <option value="Simples">Simples (Valor Único)</option>
                    <option value="Composto">
                      Composto (combinação de campos)
                    </option>
                  </select>
                  <p className="text-xs text-slate-500 mt-1">
                    Composto (combinação de campos): são metadados que precisam
                    estar relacionados entre si para fazer sentido analítico.
                    Exemplo: o metadado valor da proposta deve estar vinculado
                    ao metadado CNPJ, de forma que seja possível identificar o
                    valor correspondente a cada proponente.
                  </p>
                </div>

                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">
                    Admite Múltiplos? <span className="text-red-500">*</span>
                  </label>
                  <select
                    {...register("admiteMultiplos")}
                    className="w-full px-4 py-2 border border-slate-300 rounded-lg bg-slate-50"
                  >
                    <option value="Sim">Sim</option>
                    <option value="Não">Não (Apenas um)</option>
                  </select>
                  <p className="text-xs text-slate-500 mt-1">
                    Marque sim, caso um mesmo metadado precise ser preenchido
                    múltiplas vezes, exemplo: o metadado CNPJ pode ocorrer mais
                    de uma vez no mesmo documento, exigindo a possibilidade de
                    registo de múltiplos valores.
                  </p>
                </div>

                <div className="md:col-span-2 grid grid-cols-1 md:grid-cols-2 gap-6">
                  <div>
                    <label className="block text-sm font-medium text-slate-700 mb-1 flex items-center gap-1">
                      Formato do Dado <span className="text-red-500">*</span>
                    </label>
                    <select
                      {...register("formatoMetadado")}
                      className={`w-full px-4 py-2 border rounded-lg bg-slate-50 ${
                        errors.formatoMetadado
                          ? "border-red-500"
                          : "border-slate-300"
                      }`}
                    >
                      {formatoOpcoes.map((opt) => (
                        <option key={opt} value={opt}>
                          {opt}
                        </option>
                      ))}
                    </select>
                    {errors.formatoMetadado && (
                      <p className="text-xs text-red-600 mt-1">
                        {errors.formatoMetadado.message}
                      </p>
                    )}
                  </div>

                  {showExemplo && (
                    <div className="animate-in fade-in">
                      <label className="block text-sm font-medium text-slate-700 mb-1">
                        Exemplo de Valor (para validação)
                      </label>
                      <input
                        type="text"
                        {...register("exemploDeValor")}
                        placeholder={`Ex: para ${formatoMetadado}`}
                        className={`w-full px-4 py-2 border rounded-lg ${
                          errors.exemploDeValor
                            ? "border-red-500"
                            : "border-slate-300"
                        }`}
                      />
                      {errors.exemploDeValor && (
                        <p className="text-xs text-red-600 mt-1">
                          {errors.exemploDeValor.message}
                        </p>
                      )}
                    </div>
                  )}
                </div>

                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1 flex items-center gap-1">
                    Metadado de Processo?{" "}
                    <Info
                      className="h-3 w-3 text-slate-400 cursor-help outline-none"
                      title="Indica se a informação se aplica ao processo como um todo ou apenas ao documento em si."
                    />
                  </label>
                  <select
                    {...register("metadadoDeProcesso")}
                    className="w-full px-4 py-2 border border-slate-300 rounded-lg bg-slate-50"
                  >
                    <option value="Não">Não</option>
                    <option value="Sim">Sim</option>
                  </select>
                </div>

                <div className="md:col-span-2">
                  <label className="block text-sm font-medium text-slate-700 mb-1">
                    Classe Processual
                  </label>
                  <input
                    type="text"
                    placeholder="Buscar classe processual..."
                    className="w-full px-3 py-1.5 mb-2 text-sm border border-slate-300 rounded bg-white"
                    value={filtroClasse}
                    onChange={(e) => setFiltroClasse(e.target.value)}
                  />
                  <div
                    className={`h-48 overflow-y-auto border rounded-lg bg-slate-50 p-3 space-y-1 ${
                      errors.classeProcessual
                        ? "border-red-500"
                        : "border-slate-300"
                    }`}
                  >
                    {classesFiltradas.map((cp) => (
                      <label
                        key={cp}
                        className="flex items-start gap-2 text-sm cursor-pointer hover:bg-slate-100 p-1 rounded"
                      >
                        <input
                          type="checkbox"
                          value={cp}
                          {...register("classeProcessual")}
                          className="mt-1 flex-shrink-0"
                        />
                        <span className="text-slate-700">{cp}</span>
                      </label>
                    ))}
                    {classesFiltradas.length === 0 && (
                      <p className="text-xs text-slate-500 italic">
                        Nenhuma classe encontrada.
                      </p>
                    )}
                  </div>
                  <p className="text-xs text-slate-500 mt-1">
                    Selecione uma ou mais classes aplicáveis.
                  </p>
                  {errors.classeProcessual && (
                    <p className="text-xs text-red-600 mt-1">
                      {errors.classeProcessual.message}
                    </p>
                  )}

                  {watchedClasseProcessual.length > 0 && (
                    <div className="mt-3 p-3 bg-white border border-slate-200 rounded-lg">
                      <p className="text-xs font-bold text-slate-500 mb-2 uppercase tracking-wider">
                        Classes Selecionadas ({watchedClasseProcessual.length}):
                      </p>
                      <div className="flex flex-wrap gap-1.5">
                        {watchedClasseProcessual.map((cp) => (
                          <span
                            key={cp}
                            className="bg-indigo-50 border border-indigo-100 text-indigo-700 px-2 py-1 rounded text-xs font-medium"
                          >
                            {cp}
                          </span>
                        ))}
                      </div>
                    </div>
                  )}
                </div>
              </div>
            </div>

            {/* Seção 2.1: Estrutura Composta Dinâmica */}
            {tipoMetadados === "Composto" && (
              <div className="animate-in fade-in">
                <h3 className="text-sm font-bold text-amber-800 uppercase tracking-wider mb-4 pb-2 border-b border-amber-200">
                  2.1 Definição da Estrutura Composta
                </h3>
                <div className="space-y-4">
                  {fields.map((field, index) => (
                    <div
                      key={field.id}
                      className="flex items-start gap-4 p-4 bg-amber-50/50 border border-amber-200 rounded-lg"
                    >
                      <span className="text-sm font-bold text-amber-700 pt-2">
                        {index + 1}.
                      </span>
                      <div className="flex-1 grid grid-cols-1 sm:grid-cols-2 gap-4">
                        <div>
                          <label className="block text-xs font-medium text-slate-700 mb-1">
                            Nome do Sub-campo
                          </label>
                          <input
                            {...register(`estruturaComposta.${index}.nome`)}
                            placeholder="Ex: Encaminhamento"
                            className={`w-full px-3 py-2 text-sm border rounded-lg ${
                              errors.estruturaComposta?.[index]?.nome
                                ? "border-red-500"
                                : "border-slate-300"
                            }`}
                          />
                          {errors.estruturaComposta?.[index]?.nome && (
                            <p className="text-xs text-red-600 mt-1">
                              {errors.estruturaComposta[index].nome.message}
                            </p>
                          )}
                        </div>
                        <div>
                          <label className="block text-xs font-medium text-slate-700 mb-1">
                            Formato do Sub-campo
                          </label>
                          <select
                            {...register(`estruturaComposta.${index}.formato`)}
                            className="w-full px-3 py-2 text-sm border rounded-lg bg-white"
                          >
                            {FORMATOS_SUB_CAMPO.map((f) => (
                              <option key={f} value={f}>
                                {f}
                              </option>
                            ))}
                          </select>
                        </div>
                      </div>
                      <button
                        type="button"
                        onClick={() => remove(index)}
                        className="text-red-500 hover:text-red-700 p-2 mt-5 disabled:opacity-50"
                        disabled={fields.length <= 1}
                      >
                        <Trash2 className="h-4 w-4" />
                      </button>
                    </div>
                  ))}
                  {errors.estruturaComposta?.root && (
                    <p className="text-xs text-red-600 mt-1">
                      {errors.estruturaComposta.root.message}
                    </p>
                  )}
                  <button
                    type="button"
                    onClick={() => append({ nome: "", formato: "Texto" })}
                    className="flex items-center gap-2 text-sm font-bold text-blue-600 hover:text-blue-800 py-2 px-3 rounded-lg bg-blue-50 hover:bg-blue-100"
                  >
                    <PlusCircle className="h-4 w-4" /> Adicionar Sub-campo
                  </button>
                </div>
              </div>
            )}

            {/* Seção 3 */}
            <div>
              <h3 className="text-sm font-bold text-slate-800 uppercase tracking-wider mb-4 pb-2 border-b border-slate-200">
                3. Contexto e Detalhes
              </h3>
              <div className="space-y-6">
                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">
                    Tipos de Documento Vinculados{" "}
                    <span className="text-red-500">*</span>
                  </label>
                  <input
                    type="text"
                    placeholder="Buscar tipo de documento..."
                    className="w-full px-3 py-1.5 mb-2 text-sm border border-slate-300 rounded bg-white"
                    value={filtroDoc}
                    onChange={(e) => setFiltroDoc(e.target.value)}
                  />
                  <div
                    className={`h-48 overflow-y-auto border rounded-lg bg-slate-50 p-3 space-y-1 ${
                      errors.tipoDocumento
                        ? "border-red-500"
                        : "border-slate-300"
                    }`}
                  >
                    {docsFiltrados.map((td) => (
                      <label
                        key={td}
                        className="flex items-start gap-2 text-sm cursor-pointer hover:bg-slate-100 p-1 rounded"
                      >
                        <input
                          type="checkbox"
                          value={td}
                          {...register("tipoDocumento")}
                          className="mt-1 flex-shrink-0"
                        />
                        <span className="text-slate-700">{td}</span>
                      </label>
                    ))}
                    {docsFiltrados.length === 0 && (
                      <p className="text-xs text-slate-500 italic">
                        Nenhum documento encontrado.
                      </p>
                    )}
                  </div>
                  <p className="text-xs text-slate-500 mt-1">
                    Selecione um ou mais tipos de documento ativos que serão
                    vinculados a este metadado.
                  </p>
                  {errors.tipoDocumento && (
                    <p className="text-xs text-red-600 mt-1">
                      {errors.tipoDocumento.message}
                    </p>
                  )}

                  {watchedTipoDocumento.length > 0 && (
                    <div className="mt-3 p-3 bg-white border border-slate-200 rounded-lg">
                      <p className="text-xs font-bold text-slate-500 mb-2 uppercase tracking-wider">
                        Tipos Documentais Selecionados (
                        {watchedTipoDocumento.length}):
                      </p>
                      <div className="flex flex-wrap gap-1.5">
                        {watchedTipoDocumento.map((td) => (
                          <span
                            key={td}
                            className="bg-slate-100 border border-slate-200 text-slate-700 px-2 py-1 rounded text-xs font-medium"
                          >
                            {td}
                          </span>
                        ))}
                      </div>
                    </div>
                  )}
                </div>

                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">
                    Finalidade do Tipo Documental{" "}
                    <span className="text-red-500">*</span>
                  </label>
                  <textarea
                    {...register("finalidadeTipoDocumento")}
                    rows="3"
                    placeholder="Ex: Para o tipo documental “Informação”: analisar mérito de representação..."
                    className={`w-full px-4 py-2 border rounded-lg ${
                      errors.finalidadeTipoDocumento
                        ? "border-red-500"
                        : "border-slate-300"
                    }`}
                  ></textarea>
                  <p className="text-xs text-slate-500 mt-1 whitespace-pre-line">
                    A finalidade deve estar associada ao tipo documental
                    específico. Exemplos: - Para o tipo documental "Informação":
                    analisar mérito de representação, analisar concessão de
                    benefício, propor normativo, analisar pedido de acesso à
                    informação. - Para o tipo documental "Requerimento":
                    requerimento de benefício, requerimento de inclusão de
                    dependente, requerimento de averbação de tempo de serviço,
                    requerimento de acesso a processo.
                  </p>
                  {errors.finalidadeTipoDocumento && (
                    <p className="text-xs text-red-600 mt-1">
                      {errors.finalidadeTipoDocumento.message}
                    </p>
                  )}
                </div>

                <div>
                  <label className="block text-sm font-medium text-slate-700 mb-1">
                    Observações Técnicas
                  </label>
                  <textarea
                    {...register("observacao")}
                    rows="3"
                    placeholder="Ex: Nesse caso é necessário uma lista com as opções padrões para o usuário. Considerar também..."
                    className="w-full px-4 py-2 border rounded-lg border-slate-300"
                  ></textarea>
                </div>
              </div>
            </div>
          </div>

          {/* Rodapé Fixo do Formulário */}
          <div className="sticky bottom-0 bg-white p-6 border-t border-slate-200 flex justify-end gap-3 z-10 shadow-[0_-4px_6px_-1px_rgba(0,0,0,0.05)] rounded-b-xl">
            <button
              type="button"
              onClick={onCancel}
              className="px-6 py-2.5 border border-slate-300 rounded-lg text-slate-700 font-bold hover:bg-slate-50 transition-colors"
            >
              Cancelar
            </button>
            <button
              type="submit"
              disabled={isSubmitting}
              className="flex items-center gap-2 px-6 py-2.5 bg-blue-600 hover:bg-blue-700 text-white rounded-lg font-bold shadow-md disabled:bg-blue-400 disabled:cursor-not-allowed transition-colors"
            >
              {isSubmitting ? (
                <Loader2 className="h-5 w-5 animate-spin" />
              ) : (
                <Save className="h-5 w-5" />
              )}
              {isSubmitting
                ? existingData
                  ? "Atualizando..."
                  : "Salvando..."
                : existingData
                ? "Atualizar Metadado"
                : "Salvar Metadado"}
            </button>
          </div>
        </form>
      </div>
    </div>
  );
};

const CampoCompostoDetail = ({ campos, level = 0 }) => {
  if (!campos || campos.length === 0) return null;
  return (
    <div className={`pl-${level * 4} mt-2`}>
      {campos.map((campo, index) => (
        <div
          key={index}
          className="text-sm text-amber-800 border-l-2 border-amber-300 pl-3 py-1"
        >
          <span className="font-semibold">
            {campo.nome || campo.nomeCampo}:
          </span>{" "}
          {campo.formato || campo.formatoCampo}
        </div>
      ))}
    </div>
  );
};

const SidePanel = ({ item, onClose, activeTab, setActiveTab, formatDate }) => (
  <div className="fixed inset-0 z-50 flex justify-end">
    <div
      className="fixed inset-0 bg-slate-900/40 backdrop-blur-sm"
      onClick={onClose}
    ></div>
    <div className="relative w-full max-w-2xl bg-white h-full shadow-2xl flex flex-col animate-in slide-in-from-right duration-300">
      <div className="bg-gradient-to-r from-[#0f2e4a] to-[#1a4b77] px-6 py-6 text-white shrink-0">
        <div className="flex justify-between items-start mb-4">
          <div className="flex gap-2 flex-wrap">
            <span className="bg-white/20 px-2.5 py-1 rounded text-xs font-bold tracking-wider">
              ID {String(item.id).padStart(3, "0")}
            </span>
            <span className="bg-cyan-500/30 border border-cyan-400/30 px-2.5 py-1 rounded text-xs font-bold tracking-wider">
              {item.unidade}
            </span>
          </div>
          <button
            onClick={onClose}
            className="bg-white/10 hover:bg-white/20 p-1.5 rounded-lg"
          >
            <X className="h-5 w-5" />
          </button>
        </div>
        <h2 className="text-2xl font-bold leading-tight mb-2">
          {item.nomeMetadado}
        </h2>
        <div className="flex gap-4 text-sm text-cyan-100 font-medium">
          <span className="flex items-center gap-1.5">
            <Calendar className="h-4 w-4" /> {formatDate(item.timestamp)}
          </span>
          <span className="flex items-center gap-1.5">
            <Layers className="h-4 w-4" /> Estrutura{" "}
            {item.tipoMetadados || "Simples"}
          </span>
        </div>
      </div>
      <div className="flex border-b border-slate-200 px-6 shrink-0 pt-2">
        <button
          onClick={() => setActiveTab("geral")}
          className={`px-4 py-3 text-sm font-bold border-b-2 ${
            activeTab === "geral"
              ? "border-blue-600 text-blue-600"
              : "border-transparent text-slate-500 hover:text-slate-700"
          }`}
        >
          Informações Gerais
        </button>
        <button
          onClick={() => setActiveTab("tecnico")}
          className={`px-4 py-3 text-sm font-bold border-b-2 ${
            activeTab === "tecnico"
              ? "border-blue-600 text-blue-600"
              : "border-transparent text-slate-500 hover:text-slate-700"
          }`}
        >
          Detalhes Técnicos
        </button>
      </div>

      <div className="flex-1 overflow-y-auto p-6 bg-slate-50">
        {activeTab === "geral" && (
          <div className="space-y-6">
            <div className="bg-white p-5 rounded-xl border border-slate-200 shadow-sm">
              <h3 className="text-sm font-bold text-slate-400 uppercase tracking-wider mb-3">
                Objetivo do Metadado
              </h3>
              <p className="text-slate-800 leading-relaxed text-[15px]">
                {item.objetivoNaoInformado ? (
                  <span className="italic text-slate-500">
                    Objetivo não informado.
                  </span>
                ) : (
                  item.objetivoMetadado || "Não informado."
                )}
              </p>
            </div>

            {item.tipoMetadados === "Composto" &&
              item.estruturaComposta &&
              item.estruturaComposta.length > 0 && (
                <div className="bg-amber-50 border-l-4 border-amber-500 p-5 rounded-r-xl">
                  <h3 className="text-sm font-bold text-amber-800 uppercase tracking-wider mb-2">
                    Estrutura Composta Definida
                  </h3>
                  <CampoCompostoDetail campos={item.estruturaComposta} />
                </div>
              )}

            <div className="bg-white p-5 rounded-xl border border-slate-200 shadow-sm">
              <h3 className="text-sm font-bold text-slate-400 uppercase tracking-wider mb-3 flex items-center gap-2">
                <FileText className="h-4 w-4" /> Tipos de Documento Vinculados
              </h3>
              <div className="flex flex-wrap gap-2">
                {item.tipoDocumento && item.tipoDocumento.length > 0 ? (
                  item.tipoDocumento.map((doc, i) => (
                    <span
                      key={i}
                      className="bg-slate-100 border border-slate-200 text-slate-700 px-3 py-1.5 rounded-lg text-sm font-medium"
                    >
                      {doc}
                    </span>
                  ))
                ) : (
                  <span className="text-slate-400 italic">
                    Nenhum especificado.
                  </span>
                )}
              </div>
            </div>

            {item.finalidadeTipoDocumento && (
              <div className="bg-emerald-50 border-l-4 border-emerald-500 p-5 rounded-r-xl shadow-sm">
                <h3 className="text-sm font-bold text-emerald-800 uppercase tracking-wider mb-2 flex items-center gap-2">
                  <FileJson className="h-4 w-4" /> Finalidade do Documento
                </h3>
                <p className="text-emerald-900 leading-relaxed text-sm whitespace-pre-line">
                  {item.finalidadeTipoDocumento}
                </p>
              </div>
            )}

            {item.observacao && (
              <div className="bg-blue-50 border-l-4 border-blue-500 p-5 rounded-r-xl shadow-sm">
                <h3 className="text-sm font-bold text-blue-800 uppercase tracking-wider mb-2 flex items-center gap-2">
                  <Info className="h-4 w-4" /> Observações
                </h3>
                <p className="text-blue-900 leading-relaxed text-sm whitespace-pre-line">
                  {item.observacao}
                </p>
              </div>
            )}
          </div>
        )}

        {activeTab === "tecnico" && (
          <div className="space-y-6">
            <div className="grid grid-cols-2 gap-4">
              <div className="bg-white p-4 rounded-xl border border-slate-200 shadow-sm">
                <p className="text-xs font-bold text-slate-400 uppercase mb-1">
                  Formato
                </p>
                <p className="font-semibold text-slate-800">
                  {item.formatoMetadado || "N/A"}
                </p>
              </div>

              <div className="bg-white p-4 rounded-xl border border-slate-200 shadow-sm">
                <p className="text-xs font-bold text-slate-400 uppercase mb-1">
                  De Processo?
                </p>
                <p className="font-semibold text-slate-800">
                  {item.metadadoDeProcesso === true ||
                  item.metadadoDeProcesso === "Sim"
                    ? "Sim"
                    : "Não"}
                </p>
              </div>

              <div className="bg-white p-4 rounded-xl border border-slate-200 shadow-sm col-span-2">
                <p className="text-xs font-bold text-slate-400 uppercase mb-1">
                  Múltiplos?
                </p>
                <p className="font-semibold text-slate-800">
                  {item.admiteMultiplos ? "Sim" : "Não"}
                </p>
              </div>

              <div className="bg-white p-4 rounded-xl border border-slate-200 shadow-sm col-span-2">
                <p className="text-xs font-bold text-slate-400 uppercase mb-2">
                  Classes Processuais Vinculadas
                </p>
                <div className="flex flex-wrap gap-1.5">
                  {item.classeProcessual && item.classeProcessual.length > 0 ? (
                    item.classeProcessual.map((cp, i) => (
                      <span
                        key={i}
                        className="bg-indigo-50 border border-indigo-100 text-indigo-700 px-2 py-1 rounded text-xs font-medium"
                      >
                        {cp}
                      </span>
                    ))
                  ) : (
                    <span className="text-slate-500 text-sm">
                      Nenhuma classe especificada.
                    </span>
                  )}
                </div>
              </div>

              {item.objetivoNaoInformado && (
                <div className="bg-red-50 p-4 rounded-xl border border-red-200 shadow-sm col-span-2 flex flex-col gap-2">
                  <p className="text-sm font-bold text-red-800 flex items-center gap-2">
                    <AlertTriangle className="h-4 w-4" /> Objetivo não informado
                    originalmente
                  </p>
                </div>
              )}
            </div>
          </div>
        )}
      </div>
      <div className="p-6 bg-white border-t border-slate-200 shrink-0 flex justify-end">
        <button
          onClick={onClose}
          className="bg-slate-100 hover:bg-slate-200 text-slate-700 font-bold px-6 py-2.5 rounded-lg transition-colors"
        >
          Fechar Painel
        </button>
      </div>
    </div>
  </div>
);

// --- COMPONENTE PRINCIPAL: App ---
export default function App() {
  // Injeta Tailwind CSS automaticamente se não existir
  useEffect(() => {
    if (!document.querySelector('script[src="https://cdn.tailwindcss.com"]')) {
      const script = document.createElement("script");
      script.src = "https://cdn.tailwindcss.com";
      document.head.appendChild(script);
    }
  }, []);

  // Estados da Aplicação
  const [view, setView] = useState("dashboard"); // 'dashboard' | 'form'
  const [editingItem, setEditingItem] = useState(null);
  const [itemToDelete, setItemToDelete] = useState(null);
  const [toastMessage, setToastMessage] = useState("");
  const [isSubmitting, setIsSubmitting] = useState(false);

  // Estados do Firebase
  const [user, setUser] = useState(null);
  const [loadingDb, setLoadingDb] = useState(true);
  const [data, setData] = useState(initialData);

  // Autenticação Firebase
  useEffect(() => {
    if (!isFirebaseConfigured) return;

    const initAuth = async () => {
      try {
        await signInAnonymously(auth);
      } catch (error) {
        console.error("Erro na autenticação:", error);
      }
    };
    initAuth();

    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  // Sincronização com Firestore
  useEffect(() => {
    if (!isFirebaseConfigured) {
      setLoadingDb(false);
      setData(initialData);
      return;
    }

    if (!user) return;

    setLoadingDb(true);
    const metadadosRef = collection(db, "metadados");

    const unsubscribe = onSnapshot(
      metadadosRef,
      (snapshot) => {
        const fetchedData = snapshot.docs.map((doc) => ({
          id: doc.id,
          ...doc.data(),
        }));

        // Junta os dados iniciais com os dados do banco (sem duplicar)
        const fetchedIds = new Set(fetchedData.map((d) => String(d.id)));
        const uniqueInitialData = initialData.filter(
          (d) => !fetchedIds.has(String(d.id))
        );

        setData([...fetchedData, ...uniqueInitialData]);
        setLoadingDb(false);
      },
      (error) => {
        console.error("Erro ao carregar dados:", error);
        setData(initialData);
        setLoadingDb(false);
      }
    );

    return () => unsubscribe();
  }, [user]);

  // Função utilitária para exibir mensagens na tela
  const showToast = (msg, isError = false) => {
    setToastMessage(isError ? `Erro: ${msg}` : msg);
    setTimeout(() => setToastMessage(""), 4000);
  };

  // Estados do Dashboard
  const [searchTerm, setSearchTerm] = useState("");
  const [selectedUnidade, setSelectedUnidade] = useState("Todas");
  const [selectedTipoDocumento, setSelectedTipoDocumento] = useState("Todos");
  const [selectedItem, setSelectedItem] = useState(null);
  const [activeTab, setActiveTab] = useState("geral");
  const [sortConfig, setSortConfig] = useState({
    key: "id",
    direction: "desc",
  });

  const [currentPage, setCurrentPage] = useState(1);
  const itemsPerPage = 10;

  // --- LÓGICA DO DASHBOARD ---
  const unidades = useMemo(() => {
    const un = new Set(data.map((item) => item.unidade || "Desconhecida"));
    return ["Todas", ...Array.from(un).sort()];
  }, [data]);

  const kpis = useMemo(
    () => ({
      total: data.length,
      processo: data.filter(
        (d) => d.metadadoDeProcesso === true || d.metadadoDeProcesso === "Sim"
      ).length,
      compostos: data.filter((d) => d.tipoMetadados === "Composto").length,
      unidades: unidades.length > 1 ? unidades.length - 1 : 0,
    }),
    [data, unidades]
  );

  const handleSort = (key) => {
    let direction = "asc";
    if (sortConfig.key === key && sortConfig.direction === "asc")
      direction = "desc";
    setSortConfig({ key, direction });
  };

  const processedData = useMemo(() => {
    let filtered = data.filter((item) => {
      const nome = String(item.nomeMetadado || "").toLowerCase();
      const objetivo = String(item.objetivoMetadado || "").toLowerCase();
      const termo = searchTerm.toLowerCase();

      const matchesSearch = nome.includes(termo) || objetivo.includes(termo);
      const matchesUnidade =
        selectedUnidade === "Todas" || item.unidade === selectedUnidade;
      const matchesTipoDoc =
        selectedTipoDocumento === "Todos" ||
        (item.tipoDocumento &&
          Array.isArray(item.tipoDocumento) &&
          item.tipoDocumento.includes(selectedTipoDocumento)) ||
        item.tipoDocumento === selectedTipoDocumento;

      return matchesSearch && matchesUnidade && matchesTipoDoc;
    });

    return filtered.sort((a, b) => {
      const valA = a[sortConfig.key] || "";
      const valB = b[sortConfig.key] || "";
      if (valA < valB) return sortConfig.direction === "asc" ? -1 : 1;
      if (valA > valB) return sortConfig.direction === "asc" ? 1 : -1;
      return 0;
    });
  }, [data, searchTerm, selectedUnidade, selectedTipoDocumento, sortConfig]);

  const paginatedData = useMemo(() => {
    const startIndex = (currentPage - 1) * itemsPerPage;
    return processedData.slice(startIndex, startIndex + itemsPerPage);
  }, [processedData, currentPage]);

  const totalPages = Math.max(
    1,
    Math.ceil(processedData.length / itemsPerPage)
  );

  const formatDate = (timestamp) => {
    if (!timestamp) return "N/A";
    try {
      return new Date(timestamp).toLocaleDateString("pt-BR", {
        day: "2-digit",
        month: "2-digit",
        year: "numeric",
        hour: "2-digit",
        minute: "2-digit",
      });
    } catch (e) {
      return "Data inválida";
    }
  };

  // --- LÓGICA CRUD (FIREBASE CLOUD) ---
  const handleFormSubmit = async (formData, id) => {
    setIsSubmitting(true);

    const entryData = {
      ...formData,
      admiteMultiplos: formData.admiteMultiplos === "Sim",
      metadadoDeProcesso: formData.metadadoDeProcesso === "Sim",
      timestamp: new Date().getTime(),
    };

    delete entryData.exemploDeValor;

    if (entryData.tipoMetadados !== "Composto") {
      delete entryData.estruturaComposta;
    }

    // Limpeza dos undefined para o Firestore não dar erro
    Object.keys(entryData).forEach((key) => {
      if (entryData[key] === undefined) {
        delete entryData[key];
      }
    });

    try {
      if (isFirebaseConfigured && user) {
        // Cria a promessa de salvar no banco
        const savePromise = id
          ? setDoc(doc(db, "metadados", String(id)), entryData, { merge: true })
          : addDoc(collection(db, "metadados"), entryData);

        // Cria um cronómetro de segurança de 8 segundos
        const timeoutPromise = new Promise((_, reject) =>
          setTimeout(() => reject(new Error("TIMEOUT_FIREBASE")), 8000)
        );

        // Tenta salvar, mas se a rede bloquear e demorar mais de 8s, ele avisa
        await Promise.race([savePromise, timeoutPromise]);

        showToast("Guardado na nuvem com sucesso!");
      } else {
        // Salva localmente se não estiver logado
        if (id) {
          setData((prev) =>
            prev.map((item) =>
              item.id === id ? { ...item, ...entryData } : item
            )
          );
        } else {
          setData((prev) => [{ id: Date.now(), ...entryData }, ...prev]);
        }
        showToast("Guardado localmente.");
      }
      setView("dashboard");
      setEditingItem(null);
    } catch (error) {
      console.error("Erro ao salvar:", error);

      // Se deu timeout (bloqueio de rede ou do CodeSandbox)
      if (error.message === "TIMEOUT_FIREBASE") {
        showToast(
          "A rede bloqueou a conexão. Guardado apenas localmente para não perder os dados.",
          true
        );

        // Salva localmente para não perder o trabalho
        if (id) {
          setData((prev) =>
            prev.map((item) =>
              item.id === id ? { ...item, ...entryData } : item
            )
          );
        } else {
          setData((prev) => [{ id: Date.now(), ...entryData }, ...prev]);
        }
        setView("dashboard");
        setEditingItem(null);
      } else {
        showToast("Erro ao salvar no banco de dados.", true);
      }
    } finally {
      setIsSubmitting(false);
    }
  };

  const handleDeleteItem = async () => {
    if (!itemToDelete) return;

    if (initialDataIds.has(itemToDelete.id)) {
      showToast("Metadados fixos padrão não podem ser excluídos.", true);
      setItemToDelete(null);
      return;
    }

    try {
      if (isFirebaseConfigured && user) {
        const docRef = doc(db, "metadados", String(itemToDelete.id));
        await deleteDoc(docRef);
      } else {
        setData((prev) => prev.filter((i) => i.id !== itemToDelete.id));
      }
      showToast("Metadado excluído com sucesso!");
    } catch (error) {
      console.error("Erro ao excluir:", error);
      showToast("Erro ao excluir o metadado.", true);
    } finally {
      setItemToDelete(null);
    }
  };

  const handleShowForm = (item = null) => {
    setEditingItem(item);
    setView("form");
  };

  return (
    <div className="min-h-screen bg-slate-100 text-slate-800 font-sans pb-12">
      {toastMessage && (
        <div
          className={`fixed top-4 right-4 z-50 px-6 py-3 rounded-lg shadow-lg flex items-center gap-3 animate-in slide-in-from-top-5 ${
            toastMessage.includes("Erro") ? "bg-red-600" : "bg-emerald-600"
          } text-white`}
        >
          {toastMessage.includes("Erro") ? (
            <AlertTriangle className="h-5 w-5" />
          ) : (
            <CheckCircle2 className="h-5 w-5" />
          )}
          <span className="font-medium">{toastMessage}</span>
        </div>
      )}

      {!isFirebaseConfigured && (
        <div className="bg-amber-100 text-amber-800 px-4 py-2 text-center text-sm font-medium border-b border-amber-200">
          ⚠️ O Banco de Dados na nuvem ainda não está configurado. Insira as
          suas chaves no código (linha 22) para partilhar com os seus colegas.
        </div>
      )}

      <header className="bg-gradient-to-r from-[#0f2e4a] to-[#1a4b77] text-white shadow-lg sticky top-0 z-40">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-4">
          <div className="flex flex-col md:flex-row md:items-center justify-between gap-4">
            <div className="flex items-center space-x-3">
              <div className="bg-white/10 p-2 rounded-lg backdrop-blur-sm border border-white/20">
                <Database className="h-7 w-7 text-cyan-300" />
              </div>
              <div>
                <h1 className="text-xl sm:text-2xl font-bold tracking-tight text-white leading-tight">
                  Gestor de Metadados
                </h1>
                <p className="text-xs sm:text-sm text-cyan-100 font-medium flex items-center gap-2">
                  Tribunal de Contas do Distrito Federal
                  {loadingDb && isFirebaseConfigured ? (
                    <span className="flex items-center gap-1 text-[10px] bg-white/20 px-2 py-0.5 rounded-full">
                      <Loader2 className="h-3 w-3 animate-spin" /> A ligar...
                    </span>
                  ) : (
                    <span
                      className={`flex items-center gap-1 text-[10px] px-2 py-0.5 rounded-full text-white shadow-sm ${
                        isFirebaseConfigured
                          ? "bg-emerald-500/80"
                          : "bg-amber-500/80"
                      }`}
                    >
                      {isFirebaseConfigured ? "Nuvem Ativa" : "Offline"}
                    </span>
                  )}
                </p>
              </div>
            </div>

            <nav className="flex items-center bg-white/10 p-1 rounded-lg border border-white/20">
              <button
                onClick={() => setView("dashboard")}
                className={`flex items-center gap-2 px-4 py-2 rounded-md text-sm font-bold transition-all ${
                  view === "dashboard"
                    ? "bg-white text-[#0f2e4a] shadow-sm"
                    : "text-slate-200 hover:text-white hover:bg-white/10"
                }`}
              >
                <LayoutGrid className="h-4 w-4" /> Consultar
              </button>
              <button
                onClick={() => handleShowForm()}
                className={`flex items-center gap-2 px-4 py-2 rounded-md text-sm font-bold transition-all ${
                  view === "form"
                    ? "bg-white text-[#0f2e4a] shadow-sm"
                    : "text-slate-200 hover:text-white hover:bg-white/10"
                }`}
              >
                <PlusCircle className="h-4 w-4" /> Cadastrar Novo
              </button>
            </nav>
          </div>
        </div>
      </header>

      <main className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8 animate-in fade-in duration-300">
        {view === "dashboard" && (
          <div className="space-y-6">
            <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
              <KPIBox
                icon={Layers}
                title="Total de Registos"
                value={kpis.total}
                color="blue"
              />
              <KPIBox
                icon={FileText}
                title="Metadados de Processo"
                value={kpis.processo}
                color="indigo"
              />
              <KPIBox
                icon={Settings}
                title="Estrutura Composta"
                value={kpis.compostos}
                color="amber"
              />
              <KPIBox
                icon={BarChart3}
                title="Unidades Mapeadas"
                value={kpis.unidades}
                color="emerald"
              />
            </div>

            <div className="bg-white p-4 rounded-xl border border-slate-200 shadow-sm flex flex-col sm:flex-row gap-4 justify-between items-center">
              <div className="relative w-full sm:max-w-xs">
                <div className="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
                  <Search className="h-5 w-5 text-slate-400" />
                </div>
                <input
                  type="text"
                  className="block w-full pl-10 pr-3 py-2.5 border border-slate-300 rounded-lg bg-slate-50 placeholder-slate-400 focus:ring-2 focus:ring-blue-500 sm:text-sm transition-colors"
                  placeholder="Pesquisar..."
                  value={searchTerm}
                  onChange={(e) => {
                    setSearchTerm(e.target.value);
                    setCurrentPage(1);
                  }}
                />
              </div>

              <div className="flex flex-wrap items-center gap-3 w-full sm:w-auto">
                {(searchTerm ||
                  selectedUnidade !== "Todas" ||
                  selectedTipoDocumento !== "Todos") && (
                  <button
                    onClick={() => {
                      setSearchTerm("");
                      setSelectedUnidade("Todas");
                      setSelectedTipoDocumento("Todos");
                      setCurrentPage(1);
                    }}
                    className="text-xs text-blue-600 hover:text-blue-800 font-bold transition-colors whitespace-nowrap"
                  >
                    Limpar Filtros
                  </button>
                )}

                <div className="flex items-center gap-2 bg-slate-50 border border-slate-300 rounded-lg px-3 py-1">
                  <Filter className="h-4 w-4 text-slate-500" />
                  <select
                    className="block w-full py-1.5 text-sm bg-transparent border-none focus:ring-0 text-slate-700 font-medium cursor-pointer max-w-[150px] truncate"
                    value={selectedUnidade}
                    onChange={(e) => {
                      setSelectedUnidade(e.target.value);
                      setCurrentPage(1);
                    }}
                  >
                    {unidades.map((u) => (
                      <option key={u} value={u}>
                        {u === "Todas" ? "Todas as Unidades" : u}
                      </option>
                    ))}
                  </select>
                </div>

                <div className="flex items-center gap-2 bg-slate-50 border border-slate-300 rounded-lg px-3 py-1">
                  <Filter className="h-4 w-4 text-slate-500" />
                  <select
                    className="block w-full py-1.5 text-sm bg-transparent border-none focus:ring-0 text-slate-700 font-medium cursor-pointer max-w-[180px] truncate"
                    value={selectedTipoDocumento}
                    onChange={(e) => {
                      setSelectedTipoDocumento(e.target.value);
                      setCurrentPage(1);
                    }}
                  >
                    <option value="Todos">Todos os Tipos Documentais</option>
                    {TIPOS_DOCUMENTO_TCDF.map((td) => (
                      <option key={td} value={td}>
                        {td}
                      </option>
                    ))}
                  </select>
                </div>

                <ExportButton data={processedData} />
              </div>
            </div>

            <div className="bg-white shadow-sm rounded-xl border border-slate-200 overflow-hidden flex flex-col max-h-[70vh]">
              <div className="overflow-auto flex-1 relative">
                <table className="min-w-full divide-y divide-slate-200 relative">
                  <thead className="bg-slate-50 sticky top-0 z-10 shadow-sm outline outline-1 outline-slate-200">
                    <tr>
                      {[
                        { key: "id", label: "ID" },
                        { key: "nomeMetadado", label: "Nome do Metadado" },
                        { key: "unidade", label: "Unidade" },
                        { key: "tipoMetadados", label: "Estrutura" },
                      ].map((col) => (
                        <th
                          key={col.key}
                          onClick={() => handleSort(col.key)}
                          className="px-6 py-4 text-left text-xs font-bold text-slate-600 uppercase tracking-wider cursor-pointer hover:bg-slate-200 group select-none bg-slate-50"
                        >
                          <div className="flex items-center gap-1">
                            {col.label}
                            <ArrowUpDown
                              className={`h-3 w-3 ${
                                sortConfig.key === col.key
                                  ? "text-blue-600"
                                  : "text-slate-400"
                              }`}
                            />
                          </div>
                        </th>
                      ))}
                      <th className="px-6 py-4 text-right text-xs font-bold text-slate-600 uppercase tracking-wider bg-slate-50">
                        Ações
                      </th>
                    </tr>
                  </thead>
                  <tbody className="bg-white divide-y divide-slate-200">
                    {loadingDb && isFirebaseConfigured ? (
                      <tr>
                        <td colSpan="5" className="text-center py-16">
                          <div className="flex justify-center items-center gap-3 text-slate-500">
                            <Loader2 className="h-6 w-6 animate-spin" /> A
                            sincronizar com a Nuvem...
                          </div>
                        </td>
                      </tr>
                    ) : paginatedData.length > 0 ? (
                      paginatedData.map((item) => (
                        <tr
                          key={item.id}
                          className="hover:bg-blue-50/50 transition-colors"
                        >
                          <td className="px-6 py-4 whitespace-nowrap text-sm text-slate-500 font-mono">
                            {typeof item.id === "number"
                              ? String(item.id).padStart(3, "0")
                              : "N/A"}
                          </td>
                          <td className="px-6 py-4">
                            <div className="flex flex-col gap-1">
                              <div className="flex items-center gap-2">
                                <span
                                  className={`text-sm font-bold ${
                                    item.metadadoDeProcesso === true ||
                                    item.metadadoDeProcesso === "Sim"
                                      ? "text-blue-800"
                                      : "text-slate-900"
                                  }`}
                                >
                                  {item.nomeMetadado}
                                </span>
                                {(item.metadadoDeProcesso === true ||
                                  item.metadadoDeProcesso === "Sim") && (
                                  <span
                                    className="inline-flex items-center px-2 py-0.5 rounded text-[10px] font-bold bg-blue-100 text-blue-700 border border-blue-200 whitespace-nowrap"
                                    title="Este é um Metadado de Processo"
                                  >
                                    Processo
                                  </span>
                                )}
                              </div>
                              <div
                                className="text-xs text-slate-500 truncate max-w-[280px] mt-0.5"
                                title={item.objetivoMetadado}
                              >
                                {item.objetivoNaoInformado ? (
                                  <span className="italic">
                                    Objetivo não informado
                                  </span>
                                ) : (
                                  item.objetivoMetadado
                                )}
                              </div>
                            </div>
                          </td>
                          <td className="px-6 py-4 whitespace-nowrap">
                            <span className="inline-flex items-center px-2.5 py-1 rounded-md text-xs font-bold border bg-slate-100 text-slate-800 border-slate-200">
                              {item.unidade}
                            </span>
                          </td>
                          <td className="px-6 py-4 whitespace-nowrap">
                            <span
                              className={`inline-flex items-center gap-1.5 px-2.5 py-1 rounded-full text-xs font-medium border ${
                                item.tipoMetadados === "Composto"
                                  ? "bg-amber-50 text-amber-700 border-amber-200"
                                  : "bg-slate-100 text-slate-700 border-slate-200"
                              }`}
                            >
                              {item.tipoMetadados === "Composto" ? (
                                <Layers className="h-3 w-3" />
                              ) : (
                                <Tag className="h-3 w-3" />
                              )}{" "}
                              {item.tipoMetadados || "Simples"}
                            </span>
                          </td>
                          <td className="px-6 py-4 whitespace-nowrap text-right text-sm font-medium space-x-2">
                            <button
                              onClick={() => {
                                setSelectedItem(item);
                                setActiveTab("geral");
                              }}
                              className="inline-flex items-center gap-1.5 bg-white border border-slate-300 text-slate-700 hover:bg-slate-50 hover:text-blue-600 px-3 py-1.5 rounded-lg shadow-sm transition-colors"
                            >
                              <Eye className="h-4 w-4" /> Consultar
                            </button>
                            {!initialDataIds.has(item.id) && (
                              <>
                                <button
                                  onClick={() => handleShowForm(item)}
                                  className="inline-flex items-center gap-1.5 bg-white border border-slate-300 text-slate-700 hover:bg-slate-50 hover:text-amber-600 px-3 py-1.5 rounded-lg shadow-sm transition-colors"
                                >
                                  <Pencil className="h-4 w-4" /> Editar
                                </button>
                                <button
                                  onClick={() => setItemToDelete(item)}
                                  className="inline-flex items-center gap-1.5 bg-white border border-slate-300 text-slate-700 hover:bg-slate-50 hover:text-red-600 px-3 py-1.5 rounded-lg shadow-sm transition-colors"
                                >
                                  <Trash2 className="h-4 w-4" /> Excluir
                                </button>
                              </>
                            )}
                          </td>
                        </tr>
                      ))
                    ) : (
                      <tr>
                        <td
                          colSpan="5"
                          className="px-6 py-16 text-center text-slate-500"
                        >
                          <FileJson className="mx-auto h-12 w-12 text-slate-300 mb-4" />
                          <p className="text-lg font-medium text-slate-700">
                            Nenhum registo encontrado
                          </p>
                          <button
                            onClick={() => {
                              setSearchTerm("");
                              setSelectedUnidade("Todas");
                              setSelectedTipoDocumento("Todos");
                            }}
                            className="mt-3 text-sm text-blue-600 font-bold hover:underline"
                          >
                            Limpar filtros de pesquisa
                          </button>
                        </td>
                      </tr>
                    )}
                  </tbody>
                </table>
              </div>

              {totalPages > 1 && (
                <div className="bg-slate-50 px-6 py-4 border-t border-slate-200 flex items-center justify-between shrink-0">
                  <div className="text-sm text-slate-500">
                    Mostrando{" "}
                    <span className="font-medium text-slate-900">
                      {(currentPage - 1) * itemsPerPage + 1}
                    </span>{" "}
                    a{" "}
                    <span className="font-medium text-slate-900">
                      {Math.min(
                        currentPage * itemsPerPage,
                        processedData.length
                      )}
                    </span>{" "}
                    de{" "}
                    <span className="font-medium text-slate-900">
                      {processedData.length}
                    </span>
                  </div>
                  <div className="flex items-center gap-2">
                    <button
                      onClick={() => setCurrentPage(1)}
                      disabled={currentPage === 1}
                      className="text-xs text-slate-500 hover:text-slate-800 disabled:opacity-50 px-2 font-medium"
                    >
                      Primeira
                    </button>
                    <button
                      onClick={() => setCurrentPage((p) => Math.max(1, p - 1))}
                      disabled={currentPage === 1}
                      className="p-1 rounded-md border border-slate-300 bg-white text-slate-500 hover:bg-slate-50 disabled:opacity-50"
                    >
                      <ChevronLeft className="h-5 w-5" />
                    </button>
                    <span className="text-sm font-medium text-slate-700 px-2">
                      Página {currentPage} de {totalPages}
                    </span>
                    <button
                      onClick={() =>
                        setCurrentPage((p) => Math.min(totalPages, p + 1))
                      }
                      disabled={currentPage === totalPages}
                      className="p-1 rounded-md border border-slate-300 bg-white text-slate-500 hover:bg-slate-50 disabled:opacity-50"
                    >
                      <ChevronRight className="h-5 w-5" />
                    </button>
                    <button
                      onClick={() => setCurrentPage(totalPages)}
                      disabled={currentPage === totalPages}
                      className="text-xs text-slate-500 hover:text-slate-800 disabled:opacity-50 px-2 font-medium"
                    >
                      Última
                    </button>
                  </div>
                </div>
              )}
            </div>
          </div>
        )}

        {view === "form" && (
          <MetadataForm
            onSubmit={handleFormSubmit}
            onCancel={() => {
              setView("dashboard");
              setEditingItem(null);
            }}
            isSubmitting={isSubmitting}
            existingData={editingItem}
            showToast={showToast}
          />
        )}
      </main>

      {selectedItem && view === "dashboard" && (
        <SidePanel
          item={selectedItem}
          onClose={() => setSelectedItem(null)}
          activeTab={activeTab}
          setActiveTab={setActiveTab}
          formatDate={formatDate}
        />
      )}

      {itemToDelete && (
        <ConfirmationModal
          isOpen={!!itemToDelete}
          onClose={() => setItemToDelete(null)}
          onConfirm={handleDeleteItem}
          title="Confirmar Exclusão"
          message={`Você tem a certeza que deseja excluir o metadado "${itemToDelete.nomeMetadado}"? Esta ação não pode ser desfeita.`}
        />
      )}
    </div>
  );
}

