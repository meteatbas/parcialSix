import { ChangeDetectorRef, Component, Input, OnDestroy, OnInit, ViewChild } from '@angular/core';
import { MatPaginator } from '@angular/material/paginator';
import { MatTableDataSource } from '@angular/material/table';
import * as moment from 'moment';
import {
  AgentDashboardStatisticsFilterType,
  DashboardStatisticsFilterType,
  DashboardStatisticsRequestType,
  IDashboardEvaluatorStatistics,
  IDashboardEvaluatorTable,
  IDashboardStatistics,
  IDashboardStatisticsRequest,
  IDashboardSubjectTable,
  IResizeEvent,
  IStatistic,
  MonthlyAverageScoreDashboardStatisticsFilterType
} from 'app/pages/page-dashboard/model/dashboard-statistics.model';
import { DashboardService } from 'app/pages/page-dashboard/dashboard.service';
import { TranslateService } from '@ngx-translate/core';
import { Subscription } from 'rxjs';
import { Theme } from 'app/shared/dynamic/model/theme.enum';
import { IProcessTemplate } from 'app/shared/model/evaCore/process-template.model';
import * as shape from 'd3-shape';
import { OperatorType } from 'app/shared/provider/model/operator-type.enum';
import { EvaOperator } from 'app/shared/provider/model/eva-operator';
import { BasePageComponent } from 'app/pages/base-page.component';
import { AppConfigService } from 'app/shared/application-configuration/app-config.service';

@Component({
  selector: 'jhi-dashboard',
  templateUrl: './dashboard.component.html',
  styleUrls: ['dashboard.component.scss']
})
export class DashboardComponent extends BasePageComponent<any> implements OnInit, OnDestroy {
  displayedColumns: string[] = ['subjectName', 'releaseCount', 'averageScore', 'objectionCount'];
  displayedEvaluatorColumns: string[] = [
    'evaluatorName',
    'evaluatorReleasedCount',
    'evaluatedAgentCount',
    'givenZeroScoreCount',
    'evaluatorAverageScore'
  ];
  dataSource: MatTableDataSource<IDashboardSubjectTable>;
  evaluatorDataSource: MatTableDataSource<IDashboardEvaluatorTable>;
  evaluatorControl: boolean;
  subjectControl: boolean;
  private _selectedProcessTemplate: IProcessTemplate;
  public OperatorType = OperatorType;
  public MonthlyAverageScoreDashboardStatisticsFilterType = MonthlyAverageScoreDashboardStatisticsFilterType;
  public DashboardStatisticsFilterType = DashboardStatisticsFilterType;
  public AgentDashboardStatisticsFilterType = AgentDashboardStatisticsFilterType;

  @Input()
  operator: EvaOperator;

  @ViewChild(MatPaginator) paginator: MatPaginator;
  @ViewChild('evaluatorTablePaginator', { read: MatPaginator }) evaluatorTablePaginator: MatPaginator;

  get selectedProcessTemplate(): IProcessTemplate {
    return this._selectedProcessTemplate;
  }

  @Input()
  set selectedProcessTemplate(value: IProcessTemplate) {
    if (this._selectedProcessTemplate === value) {
      return;
    }

    this._selectedProcessTemplate = value;

    this.loadDashboard();
  }

  translateOnChangeSubscription?: Subscription;

  theme = Theme.dark;

  isLoading = false;

  refreshPeriodsInSeconds = [0, 30, 60, 120];
  selectedRefreshPeriodInSeconds = 0;
  refresherIntervalReference?: number = null;
  width: number;
  height: number;

  dashboardStatisticsRequestTypes: DashboardStatisticsRequestType[] = Object.keys(DashboardStatisticsRequestType).map(
    key => DashboardStatisticsRequestType[key]
  );

  dashboardStatisticFilterTypes: DashboardStatisticsFilterType[] = Object.keys(DashboardStatisticsFilterType).map(
    key => DashboardStatisticsFilterType[key]
  );

  agentDashboardStatisticFilterTypes: AgentDashboardStatisticsFilterType[] = Object.keys(AgentDashboardStatisticsFilterType).map(
    key => AgentDashboardStatisticsFilterType[key]
  );

  monthlyAverageScoreDashboardStatisticsFilterTypes: MonthlyAverageScoreDashboardStatisticsFilterType[] = Object.keys(
    MonthlyAverageScoreDashboardStatisticsFilterType
  ).map(key => MonthlyAverageScoreDashboardStatisticsFilterType[key]);

  selectedDashboardStatisticsRequestType = DashboardStatisticsRequestType.THIS_MONTH;
  selectedAgentDashboardStatisticFilterType = AgentDashboardStatisticsFilterType.EVALUATOR_ID;
  selectedDashboardStatisticFilterType = DashboardStatisticsFilterType.EVALUATOR_ID;
  selectedMonthlyAverageScoreDashboardStatisticFilterType = MonthlyAverageScoreDashboardStatisticsFilterType.EVALUATOR_ID;

  curveStep = shape.curveStep;

  chartScheme = 'vivid';

  dashboardStatistics?: IDashboardStatistics;
  numberCardStatistics: IStatistic[] = [];
  numberAllCardStatistics: IStatistic[] = [];
  numberPersonalCardStatistics: IStatistic[] = [];
  numberCardStatisticsSecondRow: IStatistic[] = [];
  objectionStatuses: IStatistic[] = [];
  objectionCountsByEvaluatorId: IStatistic[] = [];

  highlyScoredAgentStatistics: IStatistic[] = [];
  lowlyScoredAgentStatistics: IStatistic[] = [];

  highlyScoredAgentStatisticsByEvaluatorId: IStatistic[] = [];
  lowlyScoredAgentStatisticsByEvaluatorId: IStatistic[] = [];

  monthlyAverageGivenScores: { name: string; series: IStatistic[] }[] = [];
  monthlyAverageProcessScores: { name: string; series: IStatistic[] }[] = [];
  monthlyProcessRanks: { name: string; series: IStatistic[] }[] = [];

  aspectRatio = new Array<Number>();
  aspectRatioWidth: number;

  constructor(
    appConfigService: AppConfigService,
    private ref: ChangeDetectorRef,
    private translateService: TranslateService,
    private dashboardService: DashboardService
  ) {
    super(appConfigService, 'EvaluationForm');
    this.aspectRatio = [innerWidth / 1.05, 275];
    this.aspectRatioWidth = innerWidth / 1.25;
  }

  ngOnInit(): void {
    this.translateOnChangeSubscription = this.translateService.onLangChange.subscribe(() => {
      this.loadDashboard();
    });
  }

  ngOnDestroy(): void {
    if (this.translateOnChangeSubscription) {
      this.translateOnChangeSubscription.unsubscribe();
    }

    if (this.refresherIntervalReference != null) {
      clearInterval(this.refresherIntervalReference);
    }
  }

  loadDashboard(): void {
    if (this.isLoading || !this.selectedDashboardStatisticsRequestType || !this._selectedProcessTemplate || !this.operator) {
      return;
    }

    this.isLoading = true;

    const dashboardStatisticsRequest: IDashboardStatisticsRequest = {
      dashboardStatisticsRequestType: this.selectedDashboardStatisticsRequestType,
      processTemplateId: this._selectedProcessTemplate.id,
      operatorId: this.operator.evaId,
      operatorType: this.operator.operatorType,
      groupName: this.operator.groupName,
      otherGroupNames: this.operator.otherGroupNames
    };

    this.dashboardService.getDashboardStatistics(dashboardStatisticsRequest).subscribe(
      httpResponse => {
        if (httpResponse.body) {
          this.ref.detach();

          this.dashboardStatistics = httpResponse.body;

          if (this.operator.operatorType === OperatorType.EVALUATOR) {
            const evaluatorStatistics = this.dashboardStatistics.dashboardEvaluatorStatistics;

            this.numberPersonalCardStatistics = [];
            this.numberAllCardStatistics = [];

            this.numberPersonalCardStatistics.push({
              name: this.translateService.instant('evaGatewayApp.dashboard.averageGivenScore'),
              value: (evaluatorStatistics.averageGivenScoreByEvaluatorId || 0).toFixed(1)
            });

            this.numberPersonalCardStatistics.push({
              name: this.translateService.instant('evaGatewayApp.dashboard.totalEvaluationCount'),
              value: evaluatorStatistics.totalEvaluationCountByEvaluatorId || 0
            });

            this.numberPersonalCardStatistics.push({
              name: this.translateService.instant('evaGatewayApp.dashboard.totalEvaluatedAgentCount'),
              value: evaluatorStatistics.totalEvaluatedAgentCountByEvaluatorId || 0
            });

            this.numberAllCardStatistics.push({
              name: this.translateService.instant('evaGatewayApp.dashboard.averageGivenScore'),
              value: (evaluatorStatistics.averageProcessScore || 0).toFixed(1)
            });

            this.numberAllCardStatistics.push({
              name: this.translateService.instant('evaGatewayApp.dashboard.totalEvaluationCount'),
              value: evaluatorStatistics.totalEvaluationCount || 0
            });

            this.numberAllCardStatistics.push({
              name: this.translateService.instant('evaGatewayApp.dashboard.totalEvaluatedAgentCount'),
              value: evaluatorStatistics.totalEvaluatedAgentCount || 0
            });

            this.highlyScoredAgentStatisticsByEvaluatorId = evaluatorStatistics.highlyScoredAgentStatisticsByEvaluatorId.map(
              agentStatistic => {
                return {
                  name: agentStatistic.agentName + ' - ' + agentStatistic.agentExtension,
                  value: (agentStatistic.agentScore || 0).toFixed(1)
                };
              }
            );

            this.lowlyScoredAgentStatisticsByEvaluatorId = evaluatorStatistics.lowlyScoredAgentStatisticsByEvaluatorId.map(
              agentStatistic => {
                return {
                  name: agentStatistic.agentName + ' - ' + agentStatistic.agentExtension,
                  value: (agentStatistic.agentScore || 0).toFixed(1)
                };
              }
            );

            this.objectionCountsByEvaluatorId = Object.keys(evaluatorStatistics.objectionCountsByEvaluatorId).map(
              objectionCountsByEvaluatorId => {
                return {
                  name: this.translateService.instant('evaGatewayApp.ObjectionStatus.' + objectionCountsByEvaluatorId),
                  value: evaluatorStatistics.objectionCountsByEvaluatorId[objectionCountsByEvaluatorId]
                };
              }
            );

            this.highlyScoredAgentStatistics = evaluatorStatistics.highlyScoredAgentStatistics.map(agentStatistic => {
              return {
                name: agentStatistic.agentName + ' - ' + agentStatistic.agentExtension,
                value: (agentStatistic.agentScore || 0).toFixed(1)
              };
            });

            this.lowlyScoredAgentStatistics = evaluatorStatistics.lowlyScoredAgentStatistics.map(agentStatistic => {
              return {
                name: agentStatistic.agentName + ' - ' + agentStatistic.agentExtension,
                value: (agentStatistic.agentScore || 0).toFixed(1)
              };
            });

            this.objectionStatuses = Object.keys(evaluatorStatistics.objectionCounts).map(objectionCountsByEvaluatorId => {
              return {
                name: this.translateService.instant('evaGatewayApp.ObjectionStatus.' + objectionCountsByEvaluatorId),
                value: evaluatorStatistics.objectionCountsByEvaluatorId[objectionCountsByEvaluatorId]
              };
            });

            this.loadMonthlyAverageScoreDashboardStatisticsData(evaluatorStatistics);

            this.highlyScoredAgentStatistics = evaluatorStatistics.highlyScoredAgentStatistics.map(agentStatistic => {
              return {
                name: agentStatistic.agentName + ' - ' + agentStatistic.agentExtension,
                value: (agentStatistic.agentScore || 0).toFixed(1)
              };
            });

            this.lowlyScoredAgentStatistics = evaluatorStatistics.lowlyScoredAgentStatistics.map(agentStatistic => {
              return {
                name: agentStatistic.agentName + ' - ' + agentStatistic.agentExtension,
                value: (agentStatistic.agentScore || 0).toFixed(1)
              };
            });

            this.objectionStatuses = Object.keys(evaluatorStatistics.objectionCounts).map(objectionStatus => {
              return {
                name: this.translateService.instant('evaGatewayApp.ObjectionStatus.' + objectionStatus),
                value: evaluatorStatistics.objectionCounts[objectionStatus]
              };
            });

            this.dataSource = new MatTableDataSource<IDashboardSubjectTable>(evaluatorStatistics['dashboardSubjectTable']);
            this.dataSource.paginator = this.paginator;
            if (this.dataSource.data) {
              for (const data of this.dataSource.data) {
                data.averageScore = Number(data.averageScore.toFixed(2));
              }
            }
            this.subjectControl = this.dataSource.data != null && this.dataSource.data.length > 0;

            this.evaluatorDataSource = new MatTableDataSource<IDashboardEvaluatorTable>(evaluatorStatistics['dashboardAdminTable']);
            this.evaluatorDataSource.paginator = this.evaluatorTablePaginator;
            if (this.evaluatorDataSource.data) {
              for (const data of this.evaluatorDataSource.data) {
                data.evaluatorAverageScore = Number(data.evaluatorAverageScore.toFixed(2));
              }
            }
            this.evaluatorControl = this.evaluatorDataSource.data != null && this.evaluatorDataSource.data.length > 0;
          } else if (this.operator.operatorType === OperatorType.SUBJECT) {
            const subjectStatistics = this.dashboardStatistics.dashboardSubjectStatistics;

            this.numberCardStatistics = [];
            this.numberCardStatisticsSecondRow = [];

            this.numberCardStatistics.push({
              name: this.translateService.instant('evaGatewayApp.dashboard.averageScore'),
              value: (subjectStatistics.averageScore || 0).toFixed(1)
            });

            this.numberCardStatistics.push({
              name: this.translateService.instant('evaGatewayApp.dashboard.evaluationCount'),
              value: subjectStatistics.evaluationCount || 0
            });

            this.numberCardStatisticsSecondRow.push({
              name: this.translateService.instant('evaGatewayApp.dashboard.averageProcessScore'),
              value: (subjectStatistics.averageProcessScore || 0).toFixed(1)
            });

            this.numberCardStatisticsSecondRow.push({
              name: this.translateService.instant('evaGatewayApp.dashboard.processRank'),
              value: subjectStatistics.processRank || 0
            });

            this.objectionStatuses = Object.keys(subjectStatistics.objectionCounts).map(objectionStatus => {
              return {
                name: this.translateService.instant('evaGatewayApp.ObjectionStatus.' + objectionStatus),
                value: subjectStatistics.objectionCounts[objectionStatus]
              };
            });

            this.monthlyAverageProcessScores = [
              {
                name: this.translateService.instant('evaGatewayApp.dashboard.averageProcessScore'),
                series: Object.keys(subjectStatistics.monthlyAverageProcessScores).map(date => {
                  return {
                    name: moment(date)
                      .locale(this.translateService.currentLang)
                      .format('MMMM / YYYY'),
                    value: (subjectStatistics.monthlyAverageProcessScores[date] || 0).toFixed(1)
                  };
                })
              }
            ];

            this.monthlyProcessRanks = [
              {
                name: this.translateService.instant('evaGatewayApp.dashboard.processRank'),
                series: Object.keys(subjectStatistics.monthlyProcessRanks).map(date => {
                  return {
                    name: moment(date)
                      .locale(this.translateService.currentLang)
                      .format('MMMM / YYYY'),
                    value: subjectStatistics.monthlyProcessRanks[date] || 0
                  };
                })
              }
            ];
          }

          this.ref.reattach();
        }
        this.selectedDashboardStatisticFilterType = DashboardStatisticsFilterType.EVALUATOR_ID;
        this.selectedAgentDashboardStatisticFilterType = AgentDashboardStatisticsFilterType.EVALUATOR_ID;
        this.selectedMonthlyAverageScoreDashboardStatisticFilterType = MonthlyAverageScoreDashboardStatisticsFilterType.EVALUATOR_ID;
        this.isLoading = false;
      },
      error => {
        console.error(error);
        this.isLoading = false;
      }
    );
  }

  ofRefreshPeriodChange(newRefreshPeriodInSeconds: number): void {
    this.selectedRefreshPeriodInSeconds = newRefreshPeriodInSeconds;

    if (this.refresherIntervalReference != null) {
      window.clearInterval(this.refresherIntervalReference);
      this.refresherIntervalReference = null;
    }

    if (this.selectedRefreshPeriodInSeconds != null && this.selectedRefreshPeriodInSeconds > 0) {
      this.refresherIntervalReference = window.setInterval(() => {
        this.loadDashboard();
      }, this.selectedRefreshPeriodInSeconds * 1000);
    }
  }

  selectDashboardStatisticsRequestType(dashboardStatisticsRequestType: DashboardStatisticsRequestType): void {
    if (this.selectedDashboardStatisticsRequestType !== dashboardStatisticsRequestType) {
      this.selectedDashboardStatisticsRequestType = dashboardStatisticsRequestType;
      this.loadDashboard();
    }
  }

  getDashboardStatisticRequestTypeLabel(dashboardStatisticsRequestType: DashboardStatisticsRequestType): string {
    if (
      dashboardStatisticsRequestType === DashboardStatisticsRequestType.THIS_MONTH ||
      dashboardStatisticsRequestType === DashboardStatisticsRequestType.LAST_MONTH
    ) {
      return this.translateService.instant('evaGatewayApp.dashboard.DashboardStatisticsRequestType.' + dashboardStatisticsRequestType);
    } else if (dashboardStatisticsRequestType === DashboardStatisticsRequestType.MONTH_T_2) {
      return moment()
        .locale(this.translateService.currentLang)
        .subtract(2, 'months')
        .format('MMMM');
    } else if (dashboardStatisticsRequestType === DashboardStatisticsRequestType.MONTH_T_3) {
      return moment()
        .locale(this.translateService.currentLang)
        .subtract(3, 'months')
        .format('MMMM');
    } else if (dashboardStatisticsRequestType === DashboardStatisticsRequestType.MONTH_T_4) {
      return moment()
        .locale(this.translateService.currentLang)
        .subtract(4, 'months')
        .format('MMMM');
    }
  }

  getExportUrl(type: OperatorType): void {
    this.isLoading = true;
    const dashboardStatisticsRequest: IDashboardStatisticsRequest = {
      dashboardStatisticsRequestType: this.selectedDashboardStatisticsRequestType,
      processTemplateId: this._selectedProcessTemplate.id,
      operatorId: this.operator.evaId,
      operatorType: type,
      groupName: this.operator.groupName,
      otherGroupNames: this.operator.otherGroupNames,
      processTemplateName: this._selectedProcessTemplate.name
    };
    if (type === OperatorType.AGENT_EXPORT) {
      this.dashboardService.exportAgentDetailDashboardStatistics(dashboardStatisticsRequest).subscribe(
        httpResponse => {
          if (httpResponse) {
            this.saveAsArrayBuffer(httpResponse, type);
          }
        },
        error => {
          console.error(error);
        }
      );
    } else if (type === OperatorType.EVALUATOR_EXPORT) {
      this.dashboardService.exportEvaluatorDashboardStatistics(dashboardStatisticsRequest).subscribe(
        httpResponse => {
          if (httpResponse) {
            this.saveAsArrayBuffer(httpResponse, type);
          }
        },
        error => {
          console.error(error);
        }
      );
    } else if (type === OperatorType.EVALUATOR) {
      this.dashboardService.exportDashboardStatistics(dashboardStatisticsRequest).subscribe(
        httpResponse => {
          if (httpResponse) {
            this.saveAsArrayBuffer(httpResponse, type);
          }
        },
        error => {
          console.error(error);
        }
      );
    } else if (type === OperatorType.OBJECTION_EXPORT) {
      this.dashboardService.exportObjectionDetailDashboardStatistics(dashboardStatisticsRequest).subscribe(
        httpResponse => {
          if (httpResponse) {
            this.saveAsArrayBuffer(httpResponse, type);
          }
        },
        error => {
          console.error(error);
        }
      );
    }
  }

  saveAsArrayBuffer(data: ArrayBuffer, operatorType: OperatorType) {
    const saveByteArray = (() => {
      const a = document.createElement('a');
      document.body.appendChild(a).setAttribute('style', 'display:none');
      return (byteData: Int8Array[], name: string) => {
        const url = window.URL.createObjectURL(
          new File([new Blob([data], { type: 'application/vnd.ms-excel' })], 'report.xlsx', { type: 'application/vnd.ms-excel' })
        );
        a.href = url;
        a.download = name;
        a.click();
        window.URL.revokeObjectURL(url);
      };
    })();
    saveByteArray([new Int8Array(4096)], `${operatorType}.xlsx`);
    this.isLoading = false;
  }

  getAgentDetailStatisticsByEvaluatorIdLabel(agentDashboardStatisticsFilterType: AgentDashboardStatisticsFilterType): string {
    if (
      agentDashboardStatisticsFilterType === AgentDashboardStatisticsFilterType.ALL ||
      agentDashboardStatisticsFilterType === AgentDashboardStatisticsFilterType.EVALUATOR_ID
    ) {
      return this.translateService.instant('evaGatewayApp.dashboard.DashboardStatisticsFilterType.' + agentDashboardStatisticsFilterType);
    }
  }

  selectAgentDetailStatisticsRequestType(agentDashboardStatisticsFilterType: AgentDashboardStatisticsFilterType): void {
    if (this.selectedAgentDashboardStatisticFilterType !== agentDashboardStatisticsFilterType) {
      this.selectedAgentDashboardStatisticFilterType = agentDashboardStatisticsFilterType;
    }
  }

  getDashboardStatisticsByEvaluatorIdLabel(dashboardStatisticsFilterType: DashboardStatisticsFilterType): string {
    if (
      dashboardStatisticsFilterType === DashboardStatisticsFilterType.ALL ||
      dashboardStatisticsFilterType === DashboardStatisticsFilterType.EVALUATOR_ID
    ) {
      return this.translateService.instant('evaGatewayApp.dashboard.DashboardStatisticsFilterType.' + dashboardStatisticsFilterType);
    }
  }

  selectDashboardStatisticsByEvaluatorIdRequestType(dashboardStatisticsFilterType: DashboardStatisticsFilterType): void {
    if (this.selectedDashboardStatisticFilterType !== dashboardStatisticsFilterType) {
      this.selectedDashboardStatisticFilterType = dashboardStatisticsFilterType;
    }
  }

  getMonthlyAverageScoreDashboardStatisticsByEvaluatorIdLabel(
    monthlyAverageScoreDashboardStatisticsFilterType: MonthlyAverageScoreDashboardStatisticsFilterType
  ): string {
    if (
      monthlyAverageScoreDashboardStatisticsFilterType === MonthlyAverageScoreDashboardStatisticsFilterType.ALL ||
      monthlyAverageScoreDashboardStatisticsFilterType === MonthlyAverageScoreDashboardStatisticsFilterType.EVALUATOR_ID
    ) {
      return this.translateService.instant(
        'evaGatewayApp.dashboard.DashboardStatisticsFilterType.' + monthlyAverageScoreDashboardStatisticsFilterType
      );
    }
  }

  selectMonthlyAverageScoreDashboardStatisticsByEvaluatorIdRequestType(
    monthlyAverageScoreDashboardStatisticsFilterType: MonthlyAverageScoreDashboardStatisticsFilterType
  ): void {
    if (this.selectedMonthlyAverageScoreDashboardStatisticFilterType !== monthlyAverageScoreDashboardStatisticsFilterType) {
      this.selectedMonthlyAverageScoreDashboardStatisticFilterType = monthlyAverageScoreDashboardStatisticsFilterType;
    }
  }

  loadMonthlyAverageScoreDashboardStatisticsData(evaluatorStatistics: IDashboardEvaluatorStatistics): void {
    this.monthlyAverageGivenScores = [
      {
        name: this.translateService.instant('evaGatewayApp.dashboard.averageGivenScore'),
        series: Object.keys(evaluatorStatistics.monthlyAverageGivenScores).map(date => {
          return {
            name: moment(date)
              .locale(this.translateService.currentLang)
              .format('MMMM / YYYY'),
            value: (evaluatorStatistics.monthlyAverageGivenScores[date] || 0).toFixed(1)
          };
        })
      }
    ];

    this.monthlyAverageProcessScores = [
      {
        name: this.translateService.instant('evaGatewayApp.dashboard.averageProcessScore'),
        series: Object.keys(evaluatorStatistics.monthlyAverageProcessScores).map(date => {
          return {
            name: moment(date)
              .locale(this.translateService.currentLang)
              .format('MMMM / YYYY'),
            value: (evaluatorStatistics.monthlyAverageProcessScores[date] || 0).toFixed(1)
          };
        })
      }
    ];
  }

  onResize(event: IResizeEvent): void {
    this.aspectRatio = [event.target.innerWidth / 1.05, 275];
    this.aspectRatioWidth = event.target.innerWidth / 1.25;
  }
}
