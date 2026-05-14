# Component Patterns — ThemeData, Shared Widgets, Forms

This reference describes the reusable widget and theme patterns for the spec.
Include this content in Sections 5, 11, 12, and 14 of the generated specification.

---

## Material 3 Theme Configuration

### Color Tokens

```dart
// lib/core/theme/app_colors.dart
import 'package:flutter/material.dart';

/// Design tokens extracted from the application mockup.
/// Replace hex values with those found in MOCKUP.html CSS / inline styles.
class AppColors {
  AppColors._();

  static const primary = Color(0xFF{{PRIMARY_COLOR_HEX_NO_HASH}}); // e.g., 0xFF2271B1
  static const primaryDark = Color(0xFF{{PRIMARY_DARK_HEX_NO_HASH}});
  static const secondary = Color(0xFF{{SECONDARY_COLOR_HEX_NO_HASH}});

  static const error = Color(0xFFD63638);
  static const warning = Color(0xFFDBA617);
  static const success = Color(0xFF00A32A);
  static const info = Color(0xFF2196F3);

  static const backgroundLight = Color(0xFFF0F0F1);
  static const surfaceLight = Color(0xFFFFFFFF);
  static const textPrimaryLight = Color(0xFF1E1E1E);
  static const textSecondaryLight = Color(0xFF646970);
  static const borderLight = Color(0xFFC3C4C7);

  static const backgroundDark = Color(0xFF121212);
  static const surfaceDark = Color(0xFF1E1E1E);
  static const textPrimaryDark = Color(0xFFE0E0E0);
  static const textSecondaryDark = Color(0xFFA0A0A0);
  static const borderDark = Color(0xFF3A3A3A);
}
```

### Typography Tokens

```dart
// lib/core/theme/app_typography.dart
import 'package:flutter/material.dart';

class AppTypography {
  AppTypography._();

  /// Replace with font family from MOCKUP.html (added to pubspec.yaml fonts:)
  static const String fontFamily = '{{PRIMARY_FONT_FAMILY}}';

  static const TextTheme textTheme = TextTheme(
    displayLarge:   TextStyle(fontFamily: fontFamily, fontSize: 32, fontWeight: FontWeight.w700),
    displayMedium:  TextStyle(fontFamily: fontFamily, fontSize: 28, fontWeight: FontWeight.w700),
    displaySmall:   TextStyle(fontFamily: fontFamily, fontSize: 24, fontWeight: FontWeight.w600),
    headlineLarge:  TextStyle(fontFamily: fontFamily, fontSize: 22, fontWeight: FontWeight.w600),
    headlineMedium: TextStyle(fontFamily: fontFamily, fontSize: 20, fontWeight: FontWeight.w600),
    headlineSmall:  TextStyle(fontFamily: fontFamily, fontSize: 18, fontWeight: FontWeight.w600),
    titleLarge:     TextStyle(fontFamily: fontFamily, fontSize: 16, fontWeight: FontWeight.w600),
    titleMedium:    TextStyle(fontFamily: fontFamily, fontSize: 14, fontWeight: FontWeight.w600),
    titleSmall:     TextStyle(fontFamily: fontFamily, fontSize: 12, fontWeight: FontWeight.w600),
    bodyLarge:      TextStyle(fontFamily: fontFamily, fontSize: 16, fontWeight: FontWeight.w400),
    bodyMedium:     TextStyle(fontFamily: fontFamily, fontSize: 14, fontWeight: FontWeight.w400),
    bodySmall:      TextStyle(fontFamily: fontFamily, fontSize: 12, fontWeight: FontWeight.w400),
    labelLarge:     TextStyle(fontFamily: fontFamily, fontSize: 14, fontWeight: FontWeight.w500),
    labelMedium:    TextStyle(fontFamily: fontFamily, fontSize: 12, fontWeight: FontWeight.w500),
    labelSmall:     TextStyle(fontFamily: fontFamily, fontSize: 11, fontWeight: FontWeight.w500),
  );
}
```

### ThemeData

```dart
// lib/core/theme/app_theme.dart
import 'package:flutter/material.dart';

import 'app_colors.dart';
import 'app_typography.dart';

class AppTheme {
  AppTheme._();

  static const double _radius = {{BORDER_RADIUS_BASE}};

  static ThemeData light() {
    final scheme = ColorScheme.fromSeed(
      seedColor: AppColors.primary,
      brightness: Brightness.light,
      error: AppColors.error,
      surface: AppColors.surfaceLight,
    );
    return _buildTheme(scheme);
  }

  static ThemeData dark() {
    final scheme = ColorScheme.fromSeed(
      seedColor: AppColors.primary,
      brightness: Brightness.dark,
      error: AppColors.error,
      surface: AppColors.surfaceDark,
    );
    return _buildTheme(scheme);
  }

  static ThemeData _buildTheme(ColorScheme scheme) {
    return ThemeData(
      useMaterial3: true,
      colorScheme: scheme,
      brightness: scheme.brightness,
      fontFamily: AppTypography.fontFamily,
      textTheme: AppTypography.textTheme.apply(
        bodyColor: scheme.onSurface,
        displayColor: scheme.onSurface,
      ),
      scaffoldBackgroundColor: scheme.surface,
      appBarTheme: AppBarTheme(
        backgroundColor: scheme.surface,
        foregroundColor: scheme.onSurface,
        elevation: 0,
        scrolledUnderElevation: 1,
        centerTitle: false,
        titleTextStyle: AppTypography.textTheme.titleLarge?.copyWith(
          color: scheme.onSurface,
        ),
      ),
      cardTheme: CardTheme(
        elevation: 1,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(_radius * 2),
        ),
        margin: EdgeInsets.zero,
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: scheme.primary,
          foregroundColor: scheme.onPrimary,
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(_radius),
          ),
          textStyle: AppTypography.textTheme.labelLarge,
        ),
      ),
      filledButtonTheme: FilledButtonThemeData(
        style: FilledButton.styleFrom(
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(_radius),
          ),
        ),
      ),
      outlinedButtonTheme: OutlinedButtonThemeData(
        style: OutlinedButton.styleFrom(
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(_radius),
          ),
        ),
      ),
      inputDecorationTheme: InputDecorationTheme(
        filled: true,
        fillColor: scheme.surfaceContainerHighest,
        contentPadding: const EdgeInsets.symmetric(horizontal: 12, vertical: 14),
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(_radius),
          borderSide: BorderSide.none,
        ),
        enabledBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(_radius),
          borderSide: BorderSide.none,
        ),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(_radius),
          borderSide: BorderSide(color: scheme.primary, width: 1.5),
        ),
        errorBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(_radius),
          borderSide: BorderSide(color: scheme.error, width: 1.5),
        ),
      ),
      chipTheme: ChipThemeData(
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(_radius * 2),
        ),
      ),
      floatingActionButtonTheme: FloatingActionButtonThemeData(
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(_radius * 2),
        ),
      ),
      navigationBarTheme: NavigationBarThemeData(
        indicatorColor: scheme.primaryContainer,
        labelTextStyle: WidgetStatePropertyAll(
          AppTypography.textTheme.labelSmall,
        ),
      ),
      dividerTheme: DividerThemeData(
        color: scheme.outlineVariant,
        thickness: 1,
        space: 1,
      ),
    );
  }
}
```

---

## Shared Widgets

### AppScaffold

```dart
// lib/shared/widgets/app_scaffold.dart
import 'package:flutter/material.dart';

class AppScaffold extends StatelessWidget {
  const AppScaffold({
    super.key,
    required this.title,
    required this.body,
    this.actions,
    this.floatingActionButton,
    this.leading,
    this.bottom,
  });

  final String title;
  final Widget body;
  final List<Widget>? actions;
  final Widget? floatingActionButton;
  final Widget? leading;
  final PreferredSizeWidget? bottom;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(title),
        leading: leading,
        actions: actions,
        bottom: bottom,
      ),
      body: body,
      floatingActionButton: floatingActionButton,
    );
  }
}
```

### LoadingIndicator

```dart
// lib/shared/widgets/loading_indicator.dart
import 'package:flutter/material.dart';

class LoadingIndicator extends StatelessWidget {
  const LoadingIndicator({super.key, this.message});
  final String? message;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          const CircularProgressIndicator(),
          if (message != null) ...[
            const SizedBox(height: 16),
            Text(message!, style: Theme.of(context).textTheme.bodyMedium),
          ],
        ],
      ),
    );
  }
}
```

### EmptyState

```dart
// lib/shared/widgets/empty_state.dart
import 'package:flutter/material.dart';

class EmptyState extends StatelessWidget {
  const EmptyState({
    super.key,
    required this.message,
    this.icon = Icons.inbox_outlined,
    this.action,
  });

  final String message;
  final IconData icon;
  final Widget? action;

  @override
  Widget build(BuildContext context) {
    final colors = Theme.of(context).colorScheme;
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(icon, size: 64, color: colors.onSurfaceVariant),
            const SizedBox(height: 12),
            Text(
              message,
              style: Theme.of(context).textTheme.bodyLarge?.copyWith(
                color: colors.onSurfaceVariant,
              ),
              textAlign: TextAlign.center,
            ),
            if (action != null) ...[
              const SizedBox(height: 16),
              action!,
            ],
          ],
        ),
      ),
    );
  }
}
```

### ConfirmDialog

```dart
// lib/shared/widgets/confirm_dialog.dart
import 'package:flutter/material.dart';

class ConfirmDialog {
  ConfirmDialog._();

  static Future<bool> show(
    BuildContext context, {
    required String title,
    required String message,
    String confirmLabel = 'Confirm',
    String cancelLabel = 'Cancel',
    bool destructive = false,
  }) async {
    final result = await showDialog<bool>(
      context: context,
      builder: (context) {
        final scheme = Theme.of(context).colorScheme;
        return AlertDialog(
          title: Text(title),
          content: Text(message),
          actions: [
            TextButton(
              onPressed: () => Navigator.of(context).pop(false),
              child: Text(cancelLabel),
            ),
            FilledButton(
              style: destructive
                  ? FilledButton.styleFrom(backgroundColor: scheme.error)
                  : null,
              onPressed: () => Navigator.of(context).pop(true),
              child: Text(confirmLabel),
            ),
          ],
        );
      },
    );
    return result ?? false;
  }
}
```

### StatusChip

```dart
// lib/shared/widgets/status_chip.dart
import 'package:flutter/material.dart';

class StatusChipConfig {
  const StatusChipConfig({required this.label, required this.color});
  final String label;
  final Color color;
}

class StatusChip extends StatelessWidget {
  const StatusChip({super.key, required this.config});
  final StatusChipConfig config;

  @override
  Widget build(BuildContext context) {
    return Chip(
      label: Text(
        config.label,
        style: Theme.of(context).textTheme.labelSmall?.copyWith(
          color: ThemeData.estimateBrightnessForColor(config.color) == Brightness.dark
              ? Colors.white
              : Colors.black87,
        ),
      ),
      backgroundColor: config.color.withValues(alpha: 0.15),
      side: BorderSide(color: config.color),
      visualDensity: VisualDensity.compact,
      materialTapTargetSize: MaterialTapTargetSize.shrinkWrap,
    );
  }
}
```

### AppNetworkImage (cached_network_image)

```dart
// lib/shared/widgets/app_network_image.dart
import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter/material.dart';

class AppNetworkImage extends StatelessWidget {
  const AppNetworkImage({
    super.key,
    required this.imageUrl,
    this.width,
    this.height,
    this.fit = BoxFit.cover,
    this.borderRadius,
  });

  final String imageUrl;
  final double? width;
  final double? height;
  final BoxFit fit;
  final BorderRadius? borderRadius;

  @override
  Widget build(BuildContext context) {
    final image = CachedNetworkImage(
      imageUrl: imageUrl,
      width: width,
      height: height,
      fit: fit,
      placeholder: (context, url) => ColoredBox(
        color: Theme.of(context).colorScheme.surfaceContainerHighest,
        child: const Center(
          child: SizedBox(
            width: 24,
            height: 24,
            child: CircularProgressIndicator(strokeWidth: 2),
          ),
        ),
      ),
      errorWidget: (context, url, error) => ColoredBox(
        color: Theme.of(context).colorScheme.surfaceContainerHighest,
        child: const Icon(Icons.broken_image_outlined),
      ),
    );
    if (borderRadius == null) return image;
    return ClipRRect(borderRadius: borderRadius!, child: image);
  }
}
```

### CachedSvgIcon (flutter_svg)

```dart
// lib/shared/widgets/cached_svg_icon.dart
import 'package:flutter/material.dart';
import 'package:flutter_svg/flutter_svg.dart';

class CachedSvgIcon extends StatelessWidget {
  const CachedSvgIcon.asset(this.path, {super.key, this.size = 24, this.color})
      : isAsset = true;
  const CachedSvgIcon.network(this.path, {super.key, this.size = 24, this.color})
      : isAsset = false;

  final String path;
  final double size;
  final Color? color;
  final bool isAsset;

  @override
  Widget build(BuildContext context) {
    final ColorFilter? filter = color == null
        ? null
        : ColorFilter.mode(color!, BlendMode.srcIn);
    if (isAsset) {
      return SvgPicture.asset(path, width: size, height: size, colorFilter: filter);
    }
    return SvgPicture.network(path, width: size, height: size, colorFilter: filter);
  }
}
```

### PullToRefreshList (pull_to_refresh)

```dart
// lib/shared/widgets/pull_to_refresh_list.dart
import 'package:flutter/material.dart';
import 'package:pull_to_refresh/pull_to_refresh.dart';

class PullToRefreshList extends StatefulWidget {
  const PullToRefreshList({
    super.key,
    required this.onRefresh,
    required this.child,
    this.onLoadMore,
  });

  final Future<void> Function() onRefresh;
  final Future<void> Function()? onLoadMore;
  final Widget child;

  @override
  State<PullToRefreshList> createState() => _PullToRefreshListState();
}

class _PullToRefreshListState extends State<PullToRefreshList> {
  late final RefreshController _controller =
      RefreshController(initialRefresh: false);

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  Future<void> _refresh() async {
    try {
      await widget.onRefresh();
      _controller.refreshCompleted();
    } catch (_) {
      _controller.refreshFailed();
    }
  }

  Future<void> _loadMore() async {
    if (widget.onLoadMore == null) {
      _controller.loadComplete();
      return;
    }
    try {
      await widget.onLoadMore!();
      _controller.loadComplete();
    } catch (_) {
      _controller.loadFailed();
    }
  }

  @override
  Widget build(BuildContext context) {
    return SmartRefresher(
      controller: _controller,
      enablePullDown: true,
      enablePullUp: widget.onLoadMore != null,
      onRefresh: _refresh,
      onLoading: _loadMore,
      header: const WaterDropHeader(),
      child: widget.child,
    );
  }
}
```

---

## Form Field Widgets

### FormTextField

```dart
// lib/shared/form/form_text_field.dart
import 'package:flutter/material.dart';

class FormTextField extends StatelessWidget {
  const FormTextField({
    super.key,
    required this.controller,
    required this.label,
    this.hint,
    this.validator,
    this.keyboardType,
    this.obscureText = false,
    this.maxLength,
    this.maxLines = 1,
    this.prefixIcon,
    this.suffixIcon,
    this.enabled = true,
  });

  final TextEditingController controller;
  final String label;
  final String? hint;
  final FormFieldValidator<String>? validator;
  final TextInputType? keyboardType;
  final bool obscureText;
  final int? maxLength;
  final int maxLines;
  final Widget? prefixIcon;
  final Widget? suffixIcon;
  final bool enabled;

  @override
  Widget build(BuildContext context) {
    return TextFormField(
      controller: controller,
      validator: validator,
      keyboardType: keyboardType,
      obscureText: obscureText,
      maxLength: maxLength,
      maxLines: obscureText ? 1 : maxLines,
      enabled: enabled,
      decoration: InputDecoration(
        labelText: label,
        hintText: hint,
        prefixIcon: prefixIcon,
        suffixIcon: suffixIcon,
        counterText: '',
      ),
    );
  }
}
```

### FormDropdownField

```dart
// lib/shared/form/form_dropdown_field.dart
import 'package:flutter/material.dart';

class DropdownOption<T> {
  const DropdownOption({required this.value, required this.label});
  final T value;
  final String label;
}

class FormDropdownField<T> extends StatelessWidget {
  const FormDropdownField({
    super.key,
    required this.value,
    required this.options,
    required this.label,
    required this.onChanged,
    this.validator,
    this.hint,
  });

  final T? value;
  final List<DropdownOption<T>> options;
  final String label;
  final ValueChanged<T?> onChanged;
  final FormFieldValidator<T>? validator;
  final String? hint;

  @override
  Widget build(BuildContext context) {
    return DropdownButtonFormField<T>(
      value: value,
      onChanged: onChanged,
      validator: validator,
      items: options
          .map((o) => DropdownMenuItem<T>(value: o.value, child: Text(o.label)))
          .toList(),
      decoration: InputDecoration(labelText: label, hintText: hint),
    );
  }
}
```

### FormDatePickerField

```dart
// lib/shared/form/form_date_picker_field.dart
import 'package:flutter/material.dart';
import 'package:intl/intl.dart';

class FormDatePickerField extends StatelessWidget {
  const FormDatePickerField({
    super.key,
    required this.controller,
    required this.label,
    this.firstDate,
    this.lastDate,
    this.validator,
  });

  final TextEditingController controller;
  final String label;
  final DateTime? firstDate;
  final DateTime? lastDate;
  final FormFieldValidator<String>? validator;

  Future<void> _pickDate(BuildContext context) async {
    final now = DateTime.now();
    final initial = controller.text.isEmpty
        ? now
        : DateTime.tryParse(controller.text) ?? now;
    final picked = await showDatePicker(
      context: context,
      initialDate: initial,
      firstDate: firstDate ?? DateTime(now.year - 50),
      lastDate: lastDate ?? DateTime(now.year + 50),
    );
    if (picked != null) {
      controller.text = DateFormat('yyyy-MM-dd').format(picked);
    }
  }

  @override
  Widget build(BuildContext context) {
    return TextFormField(
      controller: controller,
      readOnly: true,
      onTap: () => _pickDate(context),
      validator: validator,
      decoration: InputDecoration(
        labelText: label,
        suffixIcon: const Icon(Icons.calendar_today_outlined),
      ),
    );
  }
}
```

### FormSwitchField

```dart
// lib/shared/form/form_switch_field.dart
import 'package:flutter/material.dart';

class FormSwitchField extends StatelessWidget {
  const FormSwitchField({
    super.key,
    required this.value,
    required this.onChanged,
    required this.label,
    this.subtitle,
  });

  final bool value;
  final ValueChanged<bool> onChanged;
  final String label;
  final String? subtitle;

  @override
  Widget build(BuildContext context) {
    return SwitchListTile(
      value: value,
      onChanged: onChanged,
      title: Text(label),
      subtitle: subtitle != null ? Text(subtitle!) : null,
      contentPadding: EdgeInsets.zero,
    );
  }
}
```

### FormImagePickerField (if ImagePicker = yes)

```dart
// lib/shared/form/form_image_picker_field.dart
import 'dart:io';

import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';

class FormImagePickerField extends StatelessWidget {
  const FormImagePickerField({
    super.key,
    required this.imageFile,
    required this.onChanged,
    required this.label,
    this.helperText,
  });

  final File? imageFile;
  final ValueChanged<File?> onChanged;
  final String label;
  final String? helperText;

  Future<void> _pick(BuildContext context, ImageSource source) async {
    final picker = ImagePicker();
    final picked = await picker.pickImage(source: source, imageQuality: 80);
    if (picked != null) {
      onChanged(File(picked.path));
    }
  }

  @override
  Widget build(BuildContext context) {
    return InputDecorator(
      decoration: InputDecoration(labelText: label, helperText: helperText),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.stretch,
        children: [
          if (imageFile != null) ...[
            ClipRRect(
              borderRadius: BorderRadius.circular(8),
              child: Image.file(imageFile!, height: 160, fit: BoxFit.cover),
            ),
            const SizedBox(height: 8),
          ],
          Row(
            children: [
              Expanded(
                child: OutlinedButton.icon(
                  onPressed: () => _pick(context, ImageSource.camera),
                  icon: const Icon(Icons.camera_alt_outlined),
                  label: const Text('Camera'),
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: OutlinedButton.icon(
                  onPressed: () => _pick(context, ImageSource.gallery),
                  icon: const Icon(Icons.photo_library_outlined),
                  label: const Text('Gallery'),
                ),
              ),
            ],
          ),
        ],
      ),
    );
  }
}
```

---

## Form Screen Pattern

```dart
// Example: lib/features/task/presentation/task_form_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:go_router/go_router.dart';

import '../application/task_providers.dart';
import '../domain/task.dart';
import '../../../core/utils/validators.dart';
import '../../../shared/form/form_text_field.dart';
import '../../../shared/form/form_dropdown_field.dart';
import '../../../shared/form/form_date_picker_field.dart';
import '../../../shared/widgets/error_state.dart';
import '../../../shared/widgets/loading_indicator.dart';

class TaskFormScreen extends HookConsumerWidget {
  const TaskFormScreen({super.key, this.taskId});
  final String? taskId;
  bool get isEditMode => taskId != null;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final formKey = useMemoized(GlobalKey<FormState>.new);
    final titleCtrl = useTextEditingController();
    final descCtrl = useTextEditingController();
    final dueDateCtrl = useTextEditingController();
    final status = useState<TaskStatus>(TaskStatus.draft);

    final formState = ref.watch(taskFormControllerProvider);

    if (isEditMode) {
      final taskAsync = ref.watch(taskByIdProvider(taskId!));
      taskAsync.whenData((task) {
        if (titleCtrl.text.isEmpty) {
          titleCtrl.text = task.title;
          descCtrl.text = task.description ?? '';
          dueDateCtrl.text = task.dueDate.toIso8601String().split('T').first;
          status.value = task.status;
        }
      });
      if (taskAsync.isLoading) return const Scaffold(body: LoadingIndicator());
      if (taskAsync.hasError) {
        return Scaffold(body: ErrorState(error: taskAsync.error!));
      }
    }

    Future<void> submit() async {
      if (!formKey.currentState!.validate()) return;
      final controller = ref.read(taskFormControllerProvider.notifier);
      try {
        if (isEditMode) {
          await controller.update(
            taskId!,
            TaskUpdate(
              title: titleCtrl.text,
              description: descCtrl.text,
              dueDate: DateTime.parse(dueDateCtrl.text),
              status: status.value,
            ),
          );
        } else {
          await controller.create(
            TaskCreate(
              title: titleCtrl.text,
              description: descCtrl.text,
              dueDate: DateTime.parse(dueDateCtrl.text),
            ),
          );
        }
        if (context.mounted) context.pop();
      } catch (_) {
        // Error already surfaced via ref.listen in build()
      }
    }

    return Scaffold(
      appBar: AppBar(title: Text(isEditMode ? 'Edit Task' : 'New Task')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Form(
          key: formKey,
          child: Column(
            children: [
              FormTextField(
                controller: titleCtrl,
                label: 'Title',
                maxLength: 100,
                validator: Validators.compose([
                  (v) => Validators.required(v, label: 'Title'),
                  (v) => Validators.maxLength(v, 100, label: 'Title'),
                ]),
              ),
              const SizedBox(height: 16),
              FormTextField(
                controller: descCtrl,
                label: 'Description',
                maxLines: 4,
              ),
              const SizedBox(height: 16),
              FormDatePickerField(
                controller: dueDateCtrl,
                label: 'Due Date',
                validator: (v) => Validators.required(v, label: 'Due date'),
              ),
              const SizedBox(height: 16),
              FormDropdownField<TaskStatus>(
                value: status.value,
                label: 'Status',
                options: const [
                  DropdownOption(value: TaskStatus.draft, label: 'Draft'),
                  DropdownOption(value: TaskStatus.inProgress, label: 'In Progress'),
                  DropdownOption(value: TaskStatus.completed, label: 'Completed'),
                ],
                onChanged: (v) {
                  if (v != null) status.value = v;
                },
              ),
              const SizedBox(height: 24),
              SizedBox(
                width: double.infinity,
                child: FilledButton(
                  onPressed: formState.isLoading ? null : submit,
                  child: formState.isLoading
                      ? const SizedBox(
                          width: 18, height: 18,
                          child: CircularProgressIndicator(strokeWidth: 2),
                        )
                      : Text(isEditMode ? 'Update' : 'Create'),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```
